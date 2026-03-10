# embabel-agent-code

**Maven artifact:** `com.embabel.agent:embabel-agent-code`

Provides tools and domain types for building coding agents that interact with software projects — reading and writing files, running builds, executing shell commands, cloning Git repos, and inspecting JVM APIs.

---

## Package map

```
com.embabel.agent.domain.library.code/
├── SoftwareProject.kt      # Domain type for a software project
└── SymbolSearch.kt         # Symbol search result types

com.embabel.coding.tools/
├── api/
│   ├── Api.kt                       # API surface of a project
│   ├── ApiExtractor.kt              # Interface for extracting APIs
│   ├── ApiReference.kt              # Reference to an API element
│   └── ApiReferenceProvider.kt      # Spring bean interface
├── bash/
│   └── BashTools.kt                 # @Tool methods for running Bash commands
├── ci/
│   ├── Ci.kt                        # CI execution result
│   └── CiTools.kt                   # @Tool for running CI commands
├── git/
│   ├── ClonedRepositoryReference.kt # A locally cloned Git repo
│   ├── GitHubRepository.kt          # GitHub repo metadata
│   ├── GitOperations.kt             # @Tool methods for Git (clone, commit, push…)
│   └── RepositoryReferenceProvider.kt
└── jvm/
    ├── ClassGraphApiReferenceExtractor.kt  # Bytecode-level API extraction
    ├── JavaParserApiExtractor.kt           # Source-level API extraction
    └── MavenBuildSystemIntegration.kt      # @Tool for Maven builds
```

---

## Key tool classes

### `BashTools`

Exposes Bash execution as a `@Tool` method. Used by coding agents that need to run arbitrary shell commands on the project. Includes safe guards (configurable allowed commands).

### `GitOperations`

Provides tools for:
- Cloning a repository to a local directory
- Checking out branches
- Committing and pushing changes
- Showing diff / status

### `MavenBuildSystemIntegration`

Provides a `@Tool` that runs `mvn` in a specified directory and returns the build output. The coding agent uses this to verify that generated code compiles.

### `ApiExtractor`

Two implementations:
- `ClassGraphApiReferenceExtractor` — uses ClassGraph to inspect bytecode. Works on compiled projects.
- `JavaParserApiExtractor` — uses JavaParser to inspect source code. Works without compilation.

---

## `SoftwareProject`

The central domain type representing a code project. Holds the local path, build system type, language, and optionally the cloned repository reference. Actions take `SoftwareProject` as input and produce modified versions or analysis results.

---

## How to extend

### Add a new build system

1. Create a class analogous to `MavenBuildSystemIntegration` that implements the tool methods for your build system (e.g. Gradle, SBT).
2. Annotate methods with Spring AI `@Tool`.
3. Register as a Spring bean or expose as a `ToolGroup`.

### Add new coding tools

1. Create a class with `@Tool`-annotated methods.
2. Register it as a `ToolGroup` bean (or directly use it as a `toolObject` in an action).
3. Reference the tool group in your `@Action(toolGroups = [...])`.

### Add language-specific API extraction

Implement `ApiExtractor` and `ApiReferenceProvider`. Register as Spring beans. The framework discovers all `ApiReferenceProvider` beans and merges their results.
