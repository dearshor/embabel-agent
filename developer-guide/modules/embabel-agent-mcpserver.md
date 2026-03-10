# embabel-agent-mcpserver

**Maven artifact:** `com.embabel.agent:embabel-agent-mcpserver`

Exposes deployed agents as tools through the Model Context Protocol (MCP) over Server-Sent Events (SSE). This lets external clients such as Claude Desktop, the MCP Inspector, or other agents call Embabel agents as MCP tools.

---

## Package map

```
com.embabel.agent.mcpserver/
├── AbstractMcpServerConfiguration.kt   # Base Spring config for MCP server
├── McpExportToolCallbackPublisher.kt    # Publishes actions as MCP tool callbacks
├── McpServerStrategy.kt                # Strategy interface (sync vs async)
├── McpToolExport.kt                    # Describes a tool exported via MCP
├── ToolRegistry.kt                     # Registry of all exported tools
├── domain/Types.kt                     # MCP-specific domain types
├── support/extensions.kt               # Extension functions
├── support/promptUtils.kt              # Prompt format utilities
├── sync/                               # Synchronous (blocking) MCP server
│   ├── McpSyncServerStrategy.kt
│   ├── SyncToolRegistry.kt
│   ├── McpPromptPublisher.kt
│   ├── McpResourcePublisher.kt
│   └── support/PerGoalMcpExportToolCallbackPublisher.kt
└── async/                              # Asynchronous (reactive) MCP server
    ├── McpAsyncServerStrategy.kt
    ├── AsyncToolRegistry.kt
    ├── McpAsyncPromptFactory.kt
    └── support/PerGoalAsyncMcpToolExportCallbackPublisher.kt
```

---

## How it works

1. On startup, all deployed agents are scanned for `@AchievesGoal` + `export = @Export(remote = true)` actions.
2. For each exported goal, a corresponding MCP tool is registered.
3. MCP clients call the tool by name, passing the required input as JSON.
4. The server deserializes the input, creates an `AgentProcess`, runs it, and returns the result as MCP tool output.

### Sync vs Async

- **Sync** (`McpSyncServerStrategy`) — uses blocking I/O. Simpler but holds a thread for the duration of the agent run.
- **Async** (`McpAsyncServerStrategy`) — uses reactive streams. Non-blocking; required for long-running agents.

Activate via `embabel-agent-autoconfigure/embabel-agent-mcpserver-autoconfigure`.

---

## Configuring tool export

Annotate the goal-achieving action with `@Export`:

```kotlin
@AchievesGoal(
    description = "Write an amusing writeup",
    export = @Export(
        remote = true,
        name = "starNewsWriteup",
        startingInputTypes = [UserInput::class],
    )
)
@Action
fun writeup(person: StarPerson, ..., ai: Ai): Writeup = ...
```

- `remote = true` — exposes the goal as an MCP tool.
- `name` — the MCP tool name (must be globally unique).
- `startingInputTypes` — the types the MCP client must pass as input.

---

## Client configuration (Claude Desktop)

```json
{
  "mcpServers": {
    "embabel": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "http://localhost:8080/sse"]
    }
  }
}
```

---

## Extension points

### Add a custom tool publisher

Implement `McpExportToolCallbackPublisher` and register as a Spring bean to control which tools are published and how they are described.

### Add MCP prompts or resources

Implement `McpPromptPublisher` or `McpResourcePublisher` (sync) / `McpAsyncPromptPublisher` or `McpAsyncResourcePublisher` (async) and register as Spring beans.
