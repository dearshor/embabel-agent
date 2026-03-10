# embabel-agent-observability

**Maven artifact:** `com.embabel.agent:embabel-agent-observability`

Provides automatic OpenTelemetry distributed tracing and Micrometer metrics for all agent executions. No code changes are required in agent code — everything is driven by events.

> This module is written in Java (not Kotlin) because it uses Spring AOP and Micrometer APIs that have better Java ergonomics.

---

## Package map

```
com.embabel.agent.observability/
├── ObservabilityProperties.java         # embabel.observability.* config properties
├── annotation/
│   ├── Tracked.java                     # @Tracked annotation for custom spans
│   └── TrackType.java                   # Enum: PROCESSING, EXTERNAL_CALL, LLM_CALL, …
├── mdc/
│   └── MdcPropagationEventListener.java # Propagates MDC context across threads
├── metrics/
│   └── EmbabelMetricsEventListener.java # Emits Micrometer counters and timers
└── observation/
    ├── EmbabelObservationContext.java    # Carries span metadata
    ├── EmbabelFullObservationEventListener.java  # Creates OTel spans from events
    ├── EmbabelTracingObservationHandler.java     # Wraps observations in OTel spans
    ├── NonEmbabelTracingObservationHandler.java  # Handles non-Embabel observations
    ├── ChatModelObservationFilter.java   # Filters LLM metadata for tracing
    ├── ObservationKeys.java              # Span attribute key constants
    ├── ObservationUtils.java             # Helper utilities
    └── TrackedAspect.java                # AOP aspect for @Tracked
```

---

## What gets traced automatically

Without any code changes, the following are traced as spans:

| Event | Span |
|---|---|
| Agent process start/end | Root span named after the agent |
| Each action start/end | Child span named after the action |
| Every LLM call | Nested under the action span (via Spring AI) |
| Tool invocations | Nested under the LLM span |
| Planning / replanning | Separate spans |
| State transitions | Annotated on the action span |

---

## `@Tracked`

Use `@Tracked` on any Spring bean method to create a custom span:

```java
@Tracked("myOperation")
public Result doWork(Input input) {
    // automatically traced
}
```

With type and description:

```java
@Tracked(value = "callPaymentApi", type = TrackType.EXTERNAL_CALL, description = "Payment gateway")
public PaymentResult processPayment(Order order) { ... }
```

When called from within an agent action, `@Tracked` spans nest under the current action span automatically.

---

## Configuration

```yaml
embabel:
  observability:
    enabled: true
    service-name: my-agent-app

management:
  tracing:
    enabled: true
    sampling:
      probability: 1.0
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans
```

### Exporters

Embabel is exporter-agnostic. Add one dependency:

| Exporter | Dependency |
|---|---|
| Zipkin | `io.opentelemetry:opentelemetry-exporter-zipkin` |
| Langfuse | `com.quantpulsar:opentelemetry-exporter-langfuse` |
| OTLP (Jaeger, Grafana, etc.) | `io.opentelemetry:opentelemetry-exporter-otlp` |

---

## Metrics

`EmbabelMetricsEventListener` emits the following Micrometer meters:

| Meter | Type | Tags |
|---|---|---|
| `embabel.agent.process.started` | Counter | `agent` |
| `embabel.agent.process.completed` | Counter | `agent` |
| `embabel.agent.process.failed` | Counter | `agent` |
| `embabel.agent.action.started` | Counter | `agent`, `action` |
| `embabel.agent.action.completed` | Timer | `agent`, `action` |
| `embabel.agent.action.failed` | Counter | `agent`, `action` |
| `embabel.agent.llm.calls` | Counter | `agent`, `action`, `model` |

---

## MDC log correlation

`MdcPropagationEventListener` automatically sets the following MDC keys on every log statement within an agent execution:

| Key | Value |
|---|---|
| `agentProcessId` | The unique process ID |
| `agentName` | The agent's name |
| `actionName` | The currently executing action (if any) |

These are propagated across threads when parallel map operations are used.

---

## Extension points

### Add custom metrics

Implement `AgenticEventListener` and register as a Spring bean:

```java
@Component
public class CustomMetricsListener implements AgenticEventListener {
    @Override
    public void onProcessEvent(AgentProcessEvent event) {
        if (event instanceof ActionCompletedEvent e) {
            myRegistry.counter("my.custom.counter",
                "action", e.getAction().getName()
            ).increment();
        }
    }
}
```

### Add custom span attributes

Extend `EmbabelTracingObservationHandler` to add extra attributes to spans based on event payload.
