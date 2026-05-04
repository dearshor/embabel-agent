# embabel-agent-skills

**Maven artifact:** `com.embabel.agent:embabel-agent-skills`

Implements the [Agent Skills specification](https://agentskills.io/specification). Provides loading, validation, and execution of skills from local directories and GitHub repositories. Skills are exposed to LLMs as tools via the `LlmReference` programming model.

---

## Package map

```
src/main/kotlin/
├── com.embabel.agent.skills/
│   ├── Skills.kt                        # Main entry point — LlmReference implementation
│   ├── SkillFrontMatterFormatter.kt     # Interface for formatting skill metadata
│   ├── spec/
│   │   ├── SkillDefinition.kt           # Parsed skill definition (SKILL.md content)
│   │   └── SkillMetadata.kt             # Frontmatter metadata (name, description, etc.)
│   ├── support/
│   │   ├── DefaultDirectorySkillDefinitionLoader.kt  # Loads skills from local dirs
│   │   ├── DirectorySkillDefinitionLoader.kt         # Loader interface
│   │   ├── GitHubSkillDefinitionLoader.kt            # Loads skills from GitHub repos
│   │   ├── LoadedSkill.kt               # Loaded skill with instructions and resources
│   │   ├── SkillValidator.kt            # Validates skill structure and frontmatter
│   │   ├── ClaudeFrontMatterFormatter.kt # Claude-flavoured frontmatter display
│   │   ├── CursorFrontMatterFormatter.kt # Cursor-flavoured frontmatter display
│   │   └── InstructionFileReferenceExtractor.kt # Extracts file references from instructions
│   └── script/
│       ├── ScriptTool.kt                # Wraps a skill script as a Tool
│       ├── SkillScript.kt               # Parsed script metadata
│       ├── SkillScriptExecutionEngine.kt # SPI for running scripts
│       ├── ProcessSkillScriptExecutionEngine.kt  # Direct host execution
│       └── DockerSkillScriptExecutionEngine.kt   # Sandboxed Docker execution
```

---

## Key types

### `Skills`

**Package:** `com.embabel.agent.skills`

The main entry point. Implements `LlmReference` so it can be passed directly to `PromptRunner.withReferences(...)`. Provides a fluent builder API for loading skills from multiple sources:

```kotlin
val skills = Skills(name = "my-skills", description = "Skills for my agent")
    .withLocalSkills("/path/to/skills")
    .withGitHubUrl("https://github.com/anthropics/skills/tree/main/skills")
    .withScriptExecutionEngine(engine)
```

Skills supports two consumption modes:

| Mode | Method | Description |
|---|---|---|
| Bundled | Use `Skills` directly as `LlmReference` | All skills grouped under one `activate` tool |
| Individual | `skills.asIndividualReferences()` | Each skill becomes its own `LlmReference` with a per-skill activation tool |

The individual mode (`asIndividualReferences`) is preferred for most use cases — it produces one tool per skill in the LLM catalog, with per-skill names and descriptions. The LLM activates a skill by calling its tool; the instructions are returned as the tool result.

### `LoadedSkill`

**Package:** `com.embabel.agent.skills.support`

Represents a fully loaded skill — its parsed frontmatter metadata, instructions (body text), and bundled resources (scripts, references, assets). Provides:

- `getActivationText()` — the full instructions returned to the LLM on activation
- `getScriptTools(engine)` — script tools derived from the skill's `scripts/` directory
- `listResources(type)` / `readResource(type, name)` — enumerate and read bundled files

### `SkillScriptExecutionEngine`

**Package:** `com.embabel.agent.skills.script`

SPI for executing skill scripts. Two built-in implementations:

| Implementation | Description |
|---|---|
| `ProcessSkillScriptExecutionEngine` | Runs scripts directly on the host via `ProcessBuilder` |
| `DockerSkillScriptExecutionEngine` | Runs scripts in an isolated Docker container with resource limits |

Scripts receive `INPUT_DIR` and `OUTPUT_DIR` environment variables. Output files written to `OUTPUT_DIR` are collected as artifacts.

```kotlin
// Direct execution
val engine = ProcessSkillScriptExecutionEngine(
    timeout = 30.seconds,
    supportedLanguages = setOf(ScriptLanguage.PYTHON, ScriptLanguage.BASH),
)

// Sandboxed Docker execution
val engine = DockerSkillScriptExecutionEngine(
    image = "embabel/agent-sandbox:latest",
    timeout = 60.seconds,
)

// Maximum isolation (no network, limited CPU/memory)
val engine = DockerSkillScriptExecutionEngine.isolated()
```

---

## Loading sources

### Local directories

```kotlin
// Single skill (directory must contain SKILL.md)
skills.withLocalSkill("/path/to/my-skill")

// Multiple skills (scans immediate subdirectories, depth 1)
skills.withLocalSkills("/path/to/skills-parent")
```

### GitHub repositories

```kotlin
// From URL (owner, repo, branch, path inferred)
skills.withGitHubUrl("https://github.com/anthropics/skills/tree/main/skills")

// Explicit parameters
skills.withGitHubSkills(owner = "myorg", repo = "skills", skillsPath = "agent-skills")

// Non-GitHub git repos
val loader = GitHubSkillDefinitionLoader.create()
loader.fromGitUrl(url = "https://gitlab.com/myorg/skills.git", branch = "main")
```

---

## Skill directory structure

```
my-skill/
├── SKILL.md        # Required — frontmatter + instructions
├── scripts/        # Optional — executable scripts
├── references/     # Optional — reference materials
└── assets/         # Optional — images, data files, etc.
```

Skills are validated on load: frontmatter fields are checked per spec, file references in instructions must exist, and the skill name must match its parent directory name.

---

## Loop-scoped activation dedup

When consumed via `asIndividualReferences()`, each skill's activation tool uses `LoopMemo` to suppress redundant activations within the same agentic loop. If the LLM calls the activation tool a second time in the same loop iteration, it receives a short-circuit message directing it to use the instructions already present in conversation history instead of re-emitting the full skill body.

---

## Integration with PromptRunner

```kotlin
@Action
fun execute(input: UserInput, ai: Ai): Result {
    val skills = Skills("my-skills", "My skills")
        .withLocalSkills("/path/to/skills")
        .withScriptExecutionEngine(ProcessSkillScriptExecutionEngine())

    return ai.withDefaultLlm()
        .withReferences(skills.asIndividualReferences())
        .createObject(input.content)
}
```
