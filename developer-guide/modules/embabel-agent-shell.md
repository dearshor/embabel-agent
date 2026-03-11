# embabel-agent-shell

**Maven artifact:** `com.embabel.agent:embabel-agent-shell`

Provides an interactive Spring Shell CLI for interacting with deployed agents during development. It is optional and activated by the `embabel-agent-starter-shell` starter.

---

## Package map

```
com.embabel.agent.shell/
├── ShellCommands.kt                  # All shell commands (@ShellMethod)
├── TerminalServices.kt               # Terminal I/O helpers
├── DefaultPromptProvider.kt          # Default shell prompt
├── MessageGeneratorPromptProvider.kt # Prompt driven by a message generator
├── ShellProperties.kt (config/)      # embabel.agent.shell.* properties
├── ShellConfiguration.kt (config/)   # Spring configuration
├── formatProcessOutput.kt            # Formats AgentProcess results for terminal
├── markdown.kt                       # Markdown → terminal rendering
└── personality/
    ├── starwars/StarwarsPromptProvider.kt
    ├── severance/SeverancePromptProvider.kt
    ├── colossus/ColossusPromptProvider.kt
    ├── montypython/MontyPythonPromptProvider.kt
    └── hitchhiker/HitchhikerPromptProvider.kt
```

---

## Key shell commands

| Command | Method | Description |
|---|---|---|
| `execute <intent>` / `x <intent>` | `execute(...)` | Run an agent for the given natural language intent (closed or open mode) |
| `chat` | `chat()` | Enter interactive multi-turn chat |
| `agents` | `agents()` | List all deployed agents |
| `actions` | `actions()` | List all available actions |
| `goals` | `goals()` | List all available goals |
| `conditions` | `conditions()` | List all conditions |
| `choose-goal <intent>` | `chooseGoal(...)` | Show goal rankings for an intent without executing |
| `runs` | `runs()` | Show recent execution history |
| `models` | `models()` | List all registered LLM models |
| `profiles` | `profiles()` | Show active Spring profiles |
| `clear` | `clear()` | Clear the shared blackboard |

### Useful flags on `execute`

| Flag | Effect |
|---|---|
| `-p` | Show prompts |
| `-r` | Show LLM responses |
| `-o` | Open mode (use all goals/actions across all agents) |
| `-g <name>` | Force a specific goal |
| `-a <name>` | Force a specific agent |

---

## Personality providers

Each themed logging personality provides a `LoggingPersonality` bean that changes log messages. The shell prompt also changes to match the personality.

Activate via Spring profile:

```bash
--spring.profiles.active=starwars
```

To add a new personality:

1. Create a class implementing `LoggingPersonality` (and optionally `ShellPromptProvider`).
2. Annotate with `@Profile("my-profile") @Component`.
3. Place in `personality/myname/`.

---

## How `execute` works

```
ShellCommands.execute(intent, ...)
    │
    ├── Autonomy.createGoalSeeker(intent, agentPlatform)
    │       └── LLM ranks all known goals by relevance
    │
    ├── GoalChoiceApprover.approve(goalRanking)
    │       └── Rejects goals below score threshold
    │
    └── agentPlatform.runAgentFrom(selectedAgent, processOptions, bindings)
            └── Returns AgentProcess; result is formatted by formatProcessOutput()
```

The `Autonomy` object is the key entry point for mode selection. It is provided by the platform autoconfiguration.

---

## Extending the shell

Add new shell commands as a Spring `@ShellComponent` bean. Inject `AgentPlatform`, `Autonomy`, and any other dependencies normally.

```kotlin
@ShellComponent
class MyCommands(
    private val agentPlatform: AgentPlatform,
) {
    @ShellMethod("My custom command")
    fun myCommand(): String {
        return agentPlatform.agents().joinToString { it.name }
    }
}
```
