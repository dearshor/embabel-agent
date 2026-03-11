# embabel-agent-api

**Maven artifact:** `com.embabel.agent:embabel-agent-api`

This is the central module of the framework. Every agent application depends on it. It defines the programming model (annotations and DSL), the runtime types (Blackboard, AgentProcess, AgentPlatform), the planning interfaces, the chat subsystem, and the testing infrastructure.

---

## Package map

```
src/main/kotlin/
├── com.embabel.agent.api.annotation/       # @Agent, @Action, @Goal, @Condition …
│   └── support/                            # Annotation processing internals
├── com.embabel.agent.api.common/           # Ai, OperationContext, PromptRunner …
│   ├── autonomy/                           # Autonomy, GoalChoiceApprover, PlanLister
│   ├── nested/                             # ObjectCreator, TemplateOperations
│   ├── scope/                              # AgentScopeBuilder
│   ├── streaming/                          # StreamingPromptRunner
│   ├── support/                            # Delegates, BranchingAction, ScatterGather …
│   ├── thinking/                           # ThinkingPromptRunnerOperations
│   └── workflow/                           # WorkflowBuilder, ScatterGather, Feedback
├── com.embabel.agent.api.dsl/              # Kotlin DSL: agent { }, action { }
├── com.embabel.agent.api.event/            # AgenticEvent hierarchy
├── com.embabel.agent.api.reference/        # LlmReference — typed, lazy LLM object references
├── com.embabel.agent.api.tool/             # Tool types
│   └── progressive/                        # UnfoldingTool, ProgressiveTool
├── com.embabel.agent.core/                 # Core runtime types
│   ├── hitl/                               # Human-in-the-loop (Awaitable etc.)
│   ├── support/                            # DefaultAgentPlatform, SimpleAgentProcess …
│   └── deployment/                         # AgentPublisher, AgentScanningProperties
├── com.embabel.plan/                       # Planner interface + Plan, WorldState
│   ├── goap/                               # AStarGoapPlanner
│   └── utility/                            # UtilityPlanner
├── com.embabel.chat/                       # Chat session, Conversation, Chatbot
│   └── agent/                              # AgentProcessChatbot, DefaultChatAgentBuilder
├── com.embabel.ux.form/                    # Form model (Form, FormGenerator, FormBinder)
├── com.embabel.agent.test.unit/            # FakeOperationContext, FakePromptRunner
└── com.embabel.agent.spi/                  # SPI layer
    └── support/                            # AgentProcessAccessor, ExecutorAsyncer, DefaultPlannerFactory …
```

---

## Key types

### Programming model

| Type | File | Purpose |
|---|---|---|
| `@Agent` | `api/annotation/annotations.kt` | Marks a class as an agent (Spring stereotype) |
| `@Action` | `api/annotation/annotations.kt` | Marks a method as an action |
| `@Condition` | `api/annotation/annotations.kt` | Marks a method as a boolean condition |
| `@AchievesGoal` | `api/annotation/annotations.kt` | Combines `@Action` with a goal declaration |
| `@EmbabelComponent` | `api/annotation/annotations.kt` | Contributes actions/goals/conditions without being a full agent |
| `@Cost` | `api/annotation/annotations.kt` | Marks a method that returns dynamic action cost (0..1) |
| `@State` | `api/annotation/annotations.kt` | Marks a class as a flow state |
| `@RequireNameMatch` | `api/annotation/annotations.kt` | Forces binding by name on an action parameter |

### Core runtime

| Type | File | Purpose |
|---|---|---|
| `Blackboard` | `core/Blackboard.kt` | Shared append-only state for one agent execution |
| `AgentPlatform` | `core/AgentPlatform.kt` | Runs agents; manages deployed agents |
| `DefaultAgentPlatform` | `core/support/DefaultAgentPlatform.kt` | The production `@Service` implementation |
| `AgentProcess` | `core/AgentProcess.kt` | A single running execution of an agent |
| `SimpleAgentProcess` | `core/support/SimpleAgentProcess.kt` | Single-threaded process implementation |
| `ConcurrentAgentProcess` | `core/support/ConcurrentAgentProcess.kt` | Concurrent (parallel action) implementation |
| `ProcessOptions` | `core/ProcessOptions.kt` | Verbosity, identity, blackboard, tool call context, etc. |
| `AgentScope` | `core/AgentScope.kt` | A set of actions, goals, conditions |
| `Condition` | `core/Condition.kt` | Runtime representation of a condition |
| `Goal` | `core/Goal.kt` | Runtime representation of a goal |
| `Action` (core) | `core/Action.kt` | Runtime representation of an action |
| `AgentProcessAccessor` | `spi/support/AgentProcessAccessor.kt` | ThreadLocal accessor used to propagate `AgentProcess` into worker threads |
| `ExecutorAsyncer` | `spi/support/ExecutorAsyncer.kt` | `Asyncer` implementation backed by an `Executor`; propagates `AgentProcess` context to worker threads |

### AI interaction

| Type | File | Purpose |
|---|---|---|
| `Ai` | `api/common/Ai.kt` | Gateway to LLM and embedding operations |
| `PromptRunner` | `api/common/PromptRunner.kt` | Fluent builder for a single LLM prompt |
| `PromptRunner.Rendering` | `api/common/PromptRunner.kt` | Chatbot rendering context; use `respond()` for safe, exception-handling replies |
| `OperationContext` | `api/common/OperationContext.kt` | Context passed into action methods |
| `ActionContext` | `api/common/ActionContext.kt` | Extends `OperationContext` for actions |
| `LlmInteraction` | `core/support/LlmInteraction.kt` | Describes a single LLM interaction; carries `toolCallContext` for the tool loop |

#### `Rendering.respond()` — safe reply with error callback

`PromptRunner.Rendering` exposes a `respond()` overload that wraps `respondWithSystemPrompt()` in a try/catch. If the LLM call fails, the `onFailure` callback is invoked instead of propagating the exception:

```kotlin
rendering("my-template").respond(
    conversation = conversation,
    model = mapOf("user" to user),
    onFailure = { error -> AssistantMessage("Sorry, something went wrong.") },
)
```

This is the recommended approach in chatbot actions where a failed LLM call should produce a graceful user-facing message rather than crashing the session.

### Planning

| Type | File | Purpose |
|---|---|---|
| `Planner` | `plan/Planner.kt` | Generic planner interface |
| `AStarGoapPlanner` | `plan/goap/astar/AStarGoapPlanner.kt` | A\* GOAP planner |
| `UtilityPlanner` | `plan/utility/UtilityPlanner.kt` | Utility-function planner |
| `Plan` | `plan/Plan.kt` | Ordered list of steps |
| `WorldState` | `plan/WorldState.kt` | Snapshot of the world used during planning |

### DSL

| Type | File | Purpose |
|---|---|---|
| `agent {}` | `api/dsl/agent.kt` | Top-level DSL entry point |
| `AgentBuilder` | `api/dsl/AgentBuilder.kt` | Collects actions/goals/conditions from DSL lambdas |
| `TypedAgentScopeBuilder` | `api/dsl/TypedAgentScopeBuilder.kt` | Typed variant for sub-process creation |

### Chat

| Type | File | Purpose |
|---|---|---|
| `Chatbot` | `chat/Chatbot.kt` | Creates `ChatSession` objects |
| `ChatSession` | `chat/ChatSession.kt` | Single interactive session with a user |
| `Conversation` | `chat/Conversation.kt` | Ordered list of messages |
| `AgentProcessChatbot` | `chat/agent/AgentProcessChatbot.kt` | Chatbot backed by an AgentProcess |

### Tools

| Type | File | Purpose |
|---|---|---|
| `Tool` | `api/tool/Tool.kt` | Core tool interface with `call(input)` and `call(input, context)` overloads |
| `ToolCallContext` | `api/tool/ToolCallContext.kt` | Immutable, framework-agnostic context for out-of-band metadata (auth tokens, tenant IDs, correlation IDs) |
| `DelegatingTool` | `api/tool/DelegatingTool.kt` | Interface for tool decorators; provides canonical two-arg `call(input, context)` entry point |
| `MethodTool` | `api/tool/MethodTool.kt` | Wraps `@LlmTool`-annotated methods; auto-injects `ToolCallContext` parameters |
| `ToolCallContextMcpMetaConverter` | `tools/mcp/ToolCallContextMcpMetaConverter.kt` | Filters/transforms `ToolCallContext` entries before sending as MCP `_meta` to remote servers |
| `ToolResponseContentAdapter` | `spi/support/springai/ToolResponseContentAdapter.kt` | Adapts tool response content for provider-specific format requirements |
| `JsonWrappingToolResponseContentAdapter` | `spi/support/springai/ToolResponseContentAdapter.kt` | Wraps non-JSON tool responses in `{"result": "..."}` for providers like Google GenAI |
| `UnfoldingTool` | `api/tool/progressive/UnfoldingTool.kt` | A "Matryoshka" tool that exposes an outer tool; when invoked, replaces itself with a set of inner tools |
| `ProgressiveTool` | `api/tool/progressive/ProgressiveTool.kt` | Marker interface for tools that evolve during an LLM conversation |
| `LlmReference` | `api/reference/LlmReference.kt` | Lazy, typed reference to an LLM-created object; resolved on first access |

#### `ToolCallContext` — out-of-band metadata for tools

`ToolCallContext` is an immutable map-like container that carries metadata (auth tokens, tenant IDs, correlation IDs, etc.) to tools without polluting the tool's JSON input schema. It flows through the entire tool pipeline:

- **`ProcessOptions.toolCallContext`** — set at process creation to pass context to all tools in the execution.
- **`Tool.call(input, context)`** — the two-arg overload receives context explicitly. The single-arg `call(input)` delegates through `ToolCallContext.EMPTY`.
- **`DelegatingTool`** — decorator chains propagate context automatically. The two-arg `call(input, context)` is the **canonical entry point** for decorator logic; the single-arg overload routes through it. Decorators should override only the two-arg method.
- **`MethodTool`** — if an `@LlmTool`-annotated method declares a parameter of type `ToolCallContext`, the framework injects the current context automatically (the parameter is excluded from the JSON input schema sent to the LLM).
- **`SpringToolCallbackAdapter`** — bridges Spring AI's `ToolContext` with `ToolCallContext` bidirectionally.
- **`DefaultToolLoop`** — passes context to each tool call during LLM tool loop execution.

Factory methods:
- `ToolCallContext.of(mapOf("tenantId" to "acme"))` — from a map
- `ToolCallContext.of("tenantId" to "acme", "apiKey" to "secret")` — from pairs
- `ToolCallContext.EMPTY` — empty context
- `ctx1.merge(ctx2)` — merge two contexts (ctx2 wins on conflict)

#### `ToolCallContextMcpMetaConverter` — filtering context for MCP

Controls which `ToolCallContext` entries cross the process boundary to remote MCP servers. This prevents sensitive entries (API keys, auth tokens) from leaking to untrusted servers.

- `ToolCallContextMcpMetaConverter.passThrough()` — propagate all entries (trusted servers only)
- `ToolCallContextMcpMetaConverter.noOp()` — suppress all metadata
- `ToolCallContextMcpMetaConverter.allowKeys("tenantId", "correlationId")` — allowlist (recommended for production)
- `ToolCallContextMcpMetaConverter.denyKeys("apiKey", "secretToken")` — denylist
- Register a custom `@Bean` of type `ToolCallContextMcpMetaConverter` to apply globally.

#### `ToolResponseContentAdapter` — provider-specific tool response formatting

Some LLM providers require tool responses in specific formats. For example, Google GenAI requires valid JSON objects — plain text causes parse errors. `ToolResponseContentAdapter` is a per-provider strategy plugged in at configuration time:

- `ToolResponseContentAdapter.PASSTHROUGH` — default, passes content unchanged (OpenAI, Anthropic).
- `JsonWrappingToolResponseContentAdapter` — wraps non-JSON responses as `{"result": "<content>"}` (Google GenAI).

#### `Tool.ContextAwareFunction` — functional interface for context-aware tools

`Tool.of(name, description, inputSchema, function)` now accepts a `ContextAwareFunction` in addition to the existing `Function` interface. Use `ContextAwareFunction` when the tool needs access to out-of-band metadata:

```kotlin
val myTool = Tool.of("my-tool", "description", InputSchema.empty()) { input, context ->
    val tenantId = context.get<String>("tenantId") ?: "default"
    Tool.Result.text("result for $tenantId")
}
```

#### `UnfoldingTool` — `includeContextTool` parameter

`UnfoldingTool.of(...)` accepts an optional `includeContextTool: Boolean` parameter (defaults to `true`). Set it to `false` to prevent the framework context tool from being injected as one of the inner tools when the unfolding tool expands. `LlmReference` sets this to `false` internally when building its unfolding reference, so the context tool is not visible to the LLM during reference resolution.

---

### Human-in-the-loop

| Type | File | Purpose |
|---|---|---|
| `Awaitable` | `core/hitl/Awaitable.kt` | Suspend an action; resume when value arrives |
| `ConfirmationRequest` | `core/hitl/ConfirmationRequest.kt` | Request human confirmation |
| `FormBindingRequest` | `core/hitl/FormBindingRequest.kt` | Request user to fill a form |

### Events

| Type | File | Purpose |
|---|---|---|
| `AgenticEvent` | `api/event/AgenticEvent.kt` | Base event type |
| `AgentProcessEvent` | `api/event/AgentProcessEvent.kt` | Events scoped to a process |
| `AgenticEventListener` | `api/event/AgenticEventListener.kt` | Listener interface; implement and register as a bean |

---

## AgentProcess context propagation to worker threads

When actions use `Asyncer` (e.g. via `context.async { ... }` or `context.parallelMap { ... }`), the current `AgentProcess` is automatically propagated to each worker thread via `AgentProcessAccessor`:

1. Before submitting the task, `ExecutorAsyncer` captures the calling thread's `AgentProcess` via `AgentProcessAccessor.getValue()`.
2. Inside the worker thread, `AgentProcessAccessor.setValue(agentProcess)` restores the context.
3. After the block completes (or throws), `AgentProcessAccessor.reset()` cleans up the ThreadLocal to prevent stale state.

This ensures that thread-contextual operations (event publishing, logging enrichment, etc.) work correctly inside parallel sub-tasks without manual propagation.

---

## Form generation

`SimpleFormGenerator` generates UI forms from Kotlin data classes. It skips:
- Properties annotated with `@NoFormField`.
- Non-nullable constructor parameters with defaults (e.g. `timestamp: Instant = Instant.now()`), which are considered auto-populated and not user input. Nullable parameters with defaults (e.g. `bio: String? = null`) are treated as optional user fields and still appear on the form.

---

## How action methods are discovered

`AgentMetadataReader` (`api/annotation/support/AgentMetadataReader.kt`) is the entry point. It:

1. Scans a class for `@Action`, `@Goal`, `@Condition`, `@Cost` methods.
2. Uses `ActionMethodManager` to build `Action` instances from each method.
3. Resolves parameter types to infer preconditions and output types.
4. Compiles the `Agent` object that is deployed to the platform.

`DefaultActionMethodManager` (`api/annotation/support/DefaultActionMethodManager.kt`) converts a single annotated method into an `Action`. To add custom action metadata extraction logic, subclass or replace `ActionMethodManager`.

---

## How `OperationContext` flows into action methods

When an action executes, its parameters are resolved by `ActionMethodArgumentResolver` (`api/annotation/support/ActionMethodArgumentResolver.kt`):

- Domain types (e.g. `StarPerson`) are retrieved from the Blackboard by type.
- Framework types (`Ai`, `OperationContext`, `ActionContext`) are injected directly.
- `@Provided`-annotated parameters are resolved from Spring context.
- `@RequireNameMatch`-annotated parameters are resolved by exact binding name.

---

## Extending the programming model

### Add a new built-in action parameter type

1. Extend `ActionMethodArgumentResolver` to recognise the new type.
2. Register your resolver as a Spring bean of type `ActionMethodArgumentResolver`.

### Add a new planner

1. Implement `com.embabel.plan.Planner`.
2. Add a new value to `com.embabel.agent.api.common.PlannerType`.
3. Register a factory in `DefaultPlannerFactory` (`spi/support/DefaultPlannerFactory.kt`).

### Listen to agent events

```kotlin
@Component
class MyListener : AgenticEventListener {
    override fun onProcessEvent(event: AgentProcessEvent) { ... }
    override fun onPlatformEvent(event: AgentPlatformEvent) { ... }
}
```

### Implement a custom Blackboard

Implement `Blackboard` and provide a `BlackboardProvider` bean (replaces `InMemoryBlackboardProvider`).

---

## Testing

`FakeOperationContext` (`test/unit/FakeOperationContext.kt`) is the primary unit-test helper. It:

- Provides a `FakePromptRunner` that records all LLM invocations.
- Accepts pre-canned responses via `expectResponse(...)`.
- Lets you inspect what prompts were sent and what tool groups were requested.

```kotlin
val context = FakeOperationContext()
context.expectResponse(MyResult("expected"))

myAgent.someAction(input, context)

val invocation = context.llmInvocations.first()
assertTrue(invocation.prompt.contains("expected substring"))
```
