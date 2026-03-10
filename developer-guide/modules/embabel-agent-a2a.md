# embabel-agent-a2a

**Maven artifact:** `com.embabel.agent:embabel-agent-a2a`

Implements the [Google Agent-to-Agent (A2A) protocol](https://github.com/google-a2a/A2A), which is an open standard for agent interoperability. Embabel agents can be called from other A2A-enabled systems and can themselves discover and call remote A2A agents.

---

## Package map

```
com.embabel.agent.a2a.server/
├── A2ARequestHandler.kt                    # Handles incoming A2A task requests
├── AgentCardHandler.kt                     # Serves the A2A Agent Card (/.well-known/agent.json)
├── AgentSkillFactory.kt                    # Converts agent goals into A2A skills
├── config/A2AConfiguration.kt             # Spring configuration
├── events.kt                               # A2A-specific events
└── support/
    ├── A2AEndpointRegistrar.kt             # Registers /a2a endpoint
    ├── A2AStreamingHandler.kt              # Streaming A2A response support
    ├── AutonomyA2ARequestHandler.kt        # Routes requests via Autonomy
    ├── EmbabelServerGoalsAgentCardHandler.kt  # Publishes goals as A2A skills
    ├── FromGoalsAgentSkillFactory.kt       # Builds A2A skills from agent goals
    └── TaskStateManager.kt                 # Manages A2A task lifecycle
```

---

## How it works

### Serving A2A requests

1. An A2A client sends a request to `POST /a2a`.
2. `A2ARequestHandler` (specifically `AutonomyA2ARequestHandler`) receives it.
3. The request is classified to find the most suitable Embabel agent.
4. The agent runs as a normal `AgentProcess`.
5. The result is wrapped in an A2A task response and returned.

### Agent Card

The A2A Agent Card is served at `GET /a2a` (or `GET /.well-known/agent.json`). It describes:
- The agent's name and description.
- Its skills (one per exported goal).
- The input/output schema for each skill.

`EmbabelServerGoalsAgentCardHandler` builds the card from all goals that have `export = @Export(remote = true)`.

---

## Prerequisites

- Add the `embabel-agent-starter-a2a` starter dependency.
- Activate the `a2a` Spring profile.
- Set `GOOGLE_STUDIO_API_KEY` (used by the Gemini model for goal classification in A2A mode).

---

## Running the A2A web UI

```bash
docker compose --profile a2a up
```

Navigate to `http://localhost:12000/` and connect to `host.docker.internal:8080/a2a`.

---

## Extension points

### Custom request routing

Implement `A2ARequestHandler` to change how incoming A2A tasks are routed to agents.

### Custom skill factory

Implement `AgentSkillFactory` to change how agent goals are presented as A2A skills (e.g. add skill categories, change descriptions).

### Custom task state management

Implement `TaskStateManager` if you need to persist A2A task state to a database (e.g. for long-running tasks that survive restarts).
