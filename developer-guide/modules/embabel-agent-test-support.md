# embabel-agent-test-support

**Maven artifact:** `com.embabel.agent:embabel-agent-test-support` (parent POM)

This module provides helpers for unit and integration testing of agents. It lives in test scope for application code but in compile scope for framework modules that ship their own tests.

## Sub-modules

| Sub-module | Artifact | Purpose |
|---|---|---|
| `embabel-agent-test-common` | `embabel-agent-test-common` | `FakeEmbeddingModel` and other shareable test doubles |
| `embabel-agent-test-internal` | `embabel-agent-test-internal` | Internal framework test utilities, domain fixtures, `FakeAiConfiguration` |

---

## Key classes

### `FakeOperationContext`

**Location:** `embabel-agent-api/src/main/kotlin/com/embabel/agent/test/unit/FakeOperationContext.kt`

> Note: despite the path, `FakeOperationContext` is in `embabel-agent-api`, not in `embabel-agent-test-support`. This is intentional — it is part of the public API so users can import it without needing a separate test module.

The primary unit test helper. It implements `OperationContext` and provides a `FakePromptRunner` that:

- Records every LLM invocation (`llmInvocations` list).
- Returns pre-canned responses in FIFO order via `expectResponse(...)`.
- Captures the tool groups and prompt contributors that were requested.

**Java example:**

```java
var context = new FakeOperationContext();
context.expectResponse(new Writeup("Gonna be a good day"));

myAgent.writeup(starPerson, stories, horoscope, context);

var invocation = context.getLlmInvocations().getFirst();
assertTrue(invocation.getPrompt().contains("Scorpio"));
assertTrue(invocation.getInteraction().getToolGroups().isEmpty());
```

**Kotlin example:**

```kotlin
val context = FakeOperationContext()
context.expectResponse(Writeup("Gonna be a good day"))

myAgent.writeup(starPerson, stories, horoscope, context)

val invocation = context.llmInvocations.first()
assertTrue(invocation.prompt.contains("Scorpio"))
```

### `FakeEmbeddingModel`

**Location:** `embabel-agent-test-common/src/main/kotlin/com/embabel/common/test/ai/FakeEmbeddingModel.kt`

A no-op `EmbeddingModel` that returns deterministic vectors. Use in tests for RAG pipelines that need an embedding service but should not call a real API.

### `FakeAiConfiguration`

**Location:** `embabel-agent-test-internal/src/main/kotlin/com/embabel/common/test/ai/config/FakeAiConfiguration.kt`

A Spring `@Configuration` class that registers a `FakeEmbeddingModel` and minimal AI infrastructure. Use with `@SpringBootTest` or `@ContextConfiguration` for integration tests that need a full Spring context but no real LLM.

---

## Integration test utilities

`IntegrationTestUtils` (`embabel-agent-test-internal`) provides `dummyProcessContext(...)` which creates a minimal `ProcessContext` backed by a real `DefaultAgentPlatform` wired with fakes. This is the foundation for `FakeOperationContext`.

---

## Test structure recommendations

From the project coding style:

- Use `@Nested` classes to group related tests.
- Name test methods with backtick strings: `` `should create star person from user input` ``.
- In Kotlin tests, use `mockk` for mocking collaborators.
- In Java tests, use `Mockito`.
- Do not couple tests to internal implementation details (avoid accessing private fields or casting to internal classes).

```kotlin
class StarNewsFinderTest {
    private val horoscopeService = mockk<HoroscopeService>()
    private val agent = StarNewsFinder(horoscopeService, storyCount = 3)

    @Nested
    inner class `writeup action` {
        @Test
        fun `prompt must contain person name and horoscope`() {
            val context = FakeOperationContext()
            context.expectResponse(Writeup("..."))

            agent.writeup(starPerson, stories, horoscope, context)

            assertTrue(context.llmInvocations.first().prompt.contains(starPerson.name))
        }
    }
}
```
