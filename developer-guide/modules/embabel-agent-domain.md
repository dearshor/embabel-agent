# embabel-agent-domain

**Maven artifact:** `com.embabel.agent:embabel-agent-domain`

This module provides a small library of pre-built, reusable domain types that are commonly needed in agent applications. These are regular Kotlin data classes annotated with Jackson annotations for clean JSON Schema generation.

## Package

```
com.embabel.agent.domain.library/
├── Person.kt         # Person interface + SimpleNamedPerson
├── Blog.kt           # BlogPost with title, URL, summary
├── NewsStory.kt      # NewsStory with URL, headline, summary
├── Research.kt       # ResearchFindings with source list
└── Summary.kt        # Summary with text content
```

## When to use

Import these types when you need common agent-input/output shapes and don't want to define them yourself. They serve as both domain objects and LLM output targets (the LLM is asked to produce instances of these classes).

## Extending

Add new reusable domain types here that could be useful across multiple agent applications. Follow the pattern:

- Use `data class` for immutability.
- Annotate with `@JsonClassDescription` and `@JsonPropertyDescription` for LLM clarity.
- Implement relevant interfaces (e.g. `HasContent`) so the framework can extract text.

```kotlin
@JsonClassDescription("A product review from a web page")
data class ProductReview(
    val productName: String,
    @get:JsonPropertyDescription("Overall rating out of 5")
    val rating: Double,
    val summary: String,
) : HasContent {
    override val content get() = summary
}
```
