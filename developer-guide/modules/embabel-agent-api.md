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
└── com.embabel.agent.test.unit/            # FakeOperationContext, FakePromptRunner
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
| `ProcessOptions` | `core/ProcessOptions.kt` | Verbosity, identity, blackboard, etc. |
| `AgentScope` | `core/AgentScope.kt` | A set of actions, goals, conditions |
| `Condition` | `core/Condition.kt` | Runtime representation of a condition |
| `Goal` | `core/Goal.kt` | Runtime representation of a goal |
| `Action` (core) | `core/Action.kt` | Runtime representation of an action |

### AI interaction

| Type | File | Purpose |
|---|---|---|
| `Ai` | `api/common/Ai.kt` | Gateway to LLM and embedding operations |
| `PromptRunner` | `api/common/PromptRunner.kt` | Fluent builder for a single LLM prompt |
| `OperationContext` | `api/common/OperationContext.kt` | Context passed into action methods |
| `ActionContext` | `api/common/ActionContext.kt` | Extends `OperationContext` for actions |

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
