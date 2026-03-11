# Architecture Overview

## Core concepts

Embabel models agentic execution around five first-class concepts:

| Concept | Description | Representation |
|---|---|---|
| **Action** | A discrete step an agent can take. May call LLMs, tools, or plain code. | `com.embabel.agent.core.Action` |
| **Goal** | A desired outcome. An agent is finished when a goal is achieved. | `com.embabel.agent.core.Goal` |
| **Condition** | A boolean predicate evaluated from the blackboard state. Used as pre/post conditions. | `com.embabel.agent.core.Condition` |
| **Blackboard** | Shared, append-only in-memory state for a single agent execution. | `com.embabel.agent.core.Blackboard` |
| **Plan** | A dynamically computed sequence of actions towards a goal. Re-computed after each step. | `com.embabel.plan.Plan` |

## Execution model

```
User input
    │
    ▼
AgentPlatform.runAgentFrom(agent, processOptions, bindings)
    │
    ▼
AgentProcess.run()                      ← loops until goal achieved or stuck
    │
    ├── Planner.bestValuePlanToAnyGoal()  ← GOAP or Utility AI
    │       └── reads Blackboard (world state)
    │
    ├── execute next Action
    │       ├── resolve parameters from Blackboard
    │       ├── call action method (may invoke LLM via PromptRunner)
    │       └── write result back to Blackboard
    │
    └── replan
```

The OODA loop (Observe-Orient-Decide-Act) is implemented implicitly: after every action the world state is re-observed (Blackboard), a new plan is decided (Planner), and the next action is executed.

## Authoring models

### Annotation-based (Java or Kotlin)

```kotlin
@Agent(description = "...")
class MyAgent {
    @Action
    fun step1(input: InputType, ai: Ai): ResultType = ...

    @AchievesGoal(description = "...")
    @Action
    fun finalStep(result: ResultType, ai: Ai): Output = ...
}
```

The `@Agent` annotation is a Spring stereotype — the class is a regular Spring bean. Methods annotated `@Action` are discovered automatically. Parameter types drive preconditions; the return type becomes a new fact on the Blackboard.

### Kotlin DSL

```kotlin
val myAgent = agent(name = "my-agent", description = "...") {
    action<InputType, ResultType>("step1") { input, context ->
        context.ai().withDefaultLlm().createObject("prompt: $input")
    }
}
agentPlatform.deploy(myAgent)
```

Both styles produce the same `com.embabel.agent.core.Agent` object.

## Blackboard

The Blackboard is the shared state for a single `AgentProcess`. It is append-only at the object level — objects can be _hidden_ but not removed. The most recently added object of a given type is the "default" binding (`IoBinding.DEFAULT_BINDING = "it"`).

Key operations:
- `blackboard += myObject` — add an object (bound to `"it"`)
- `blackboard["name"] = myObject` — bind to a named slot
- `blackboard.last<MyType>()` — retrieve most recent object of given type
- `blackboard.objectsOfType<MyType>()` — retrieve all objects of given type

The Blackboard is accessed by the planner to determine the current world state, and by the action parameter resolver to supply inputs to action methods.

## Planning

Two planners ship out of the box:

| Planner | Class | When to use |
|---|---|---|
| **GOAP** (default) | `com.embabel.plan.goap.astar.AStarGoapPlanner` | Deterministic goal-driven flows |
| **Utility AI** | `com.embabel.plan.utility.UtilityPlanner` | Open-ended exploration, maximize value |

Select via `@Agent(planner = PlannerType.UTILITY_AI)` or `PlannerType.GOAP`.

Planners are pluggable. Implement `com.embabel.plan.Planner` to add a new strategy.

## Execution modes

| Mode | How | AgentPlatform method |
|---|---|---|
| **Focused** | Caller chooses a specific agent | `runAgentFrom(specificAgent, ...)` |
| **Closed** | Platform picks agent from those deployed, based on user intent | `Autonomy.runWithInputClassification(...)` |
| **Open** | Platform assembles a custom agent from all known goals/actions | `Autonomy.runOpenMode(...)` |

## Event system

All significant runtime events are published via `com.embabel.agent.api.event.AgenticEventListener`. Important event types:

- `AgentDeploymentEvent` — agent registered with platform
- `AgentProcessCreationEvent` — new `AgentProcess` created
- `ActionStartEvent` / `ActionCompletedEvent` / `ActionFailedEvent`
- `LlmInteractionEvent` — every LLM call
- `AgentProcessCompletedEvent` / `AgentProcessFailedEvent`

Listeners are Spring `@Component` beans that implement `AgenticEventListener`. The observability and shell modules both rely on this mechanism.

## Dependency injection and Spring integration

Agents are standard Spring beans. Dependencies are injected by Spring in the usual way. `@Value`, `@Autowired`, constructor injection — all work. This means:

- Agents can be unit-tested without Spring by constructing them with mock dependencies and a `FakeOperationContext`.
- AOP decorators (e.g. `@Transactional`, `@Retryable`) apply normally.
- Agents can persist state using Spring Data repositories.

## Human-in-the-loop (HitL)

The `com.embabel.agent.core.hitl` package provides a suspendable execution model:

- `Awaitable` — suspend an action until an external event arrives
- `ConfirmationRequest` — ask for human approval
- `FormBindingRequest` — show a form and await submission

The shell uses this to ask the user for confirmation before destructive operations.
