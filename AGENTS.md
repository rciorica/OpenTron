# AGENTS.md

## Overview & Agent Architecture

OpenTron is a privacy-first, local-first agentic AI runtime engineered around Java 21 Virtual Threads (`Project Loom`). It operates a reactive, zero-waste multi-agent ecosystem capable of coordinating over 10,000 concurrent agent tasks per single JVM process without process bloat or inter-process communication (IPC) overhead.

The codebase separates the high-concurrency Java backend/CLI from a cross-platform Tauri (React 19 + Rust) desktop interface:
---
                         ┌──────────────────────────────────────────────┐
                         │   Tauri Desktop / Web UI (React 19 + Vite)   │
                         └──────────────────────┬───────────────────────┘
                                                │ REST / WebSocket
                                                v                                         
    ┌──────────────────────────────────────────────────────────────────────────────────────────┐
    │ OpenTron JVM Runtime (Java 21 Virtual Threads)                                           │
    │                                                                                          │
    │  ┌───────────────────────┐   ┌───────────────────────────┐   ┌────────────────────────┐  │
    │  │ MultiAgentCoordinator │ ◄─┤  SkillBasedOrchestrator   ├─► │  AgentLLMBridge        │  │
    │  └───────────┬───────────┘   └─────────────┬─────────────┘   └───────────┬────────────┘  │
    │              │                             │                             │               │
    │              v                             v                             v               │
    │  ┌───────────────────────┐   ┌───────────────────────────┐   ┌────────────────────────┐  │
    │  │ Memory / Ingest Engine│   │ Connectors / Tool Registry│   │ Model Clients & Routing│  │
    │  │ (SQLite / H2 / PG)    │   │ (Gmail, Notion, Slack...) │   │  (Ollama, Anthropic...)│  │
    │  └───────────────────────┘   └─────────────┬─────────────┘   └────────────────────────┘  │
    └────────────────────────────────────────────┼─────────────────────────────────────────────┘
                                                 │
                                                 v
                                    ┌───────────────────────┐
                                    │ Sandbox & Tool Exec   │
                                    └───────────────────────┘
---

## Core Agent Components

All core backend agent execution files are located under `java/opentron-java/backend/src/main/java/org/opentron/backend/`:

### 1. Agents Engine (`/agents`)
* **`Agent.java`**: Domain model representing individual agent definitions, capabilities, and system configurations.
* **`AgentLLMBridge.java`**: Connects local/cloud LLMs (`Ollama`, `Anthropic`, `CloudModelClient`) with agent prompts and tool execution loops.
* **`AgentService.java`**: Lifecycle manager for instantiating, running, and persisting active agent instances.
* **`AgentSystemPrompts.java`**: Centralized registry for base personas and system prompts (e.g., `TRON.md`, `neutral.md`).
* **`MultiAgentCoordinator.java`**: Coordinates multi-agent workflows, delegating sub-tasks across specialized agents.
* **`SkillBasedOrchestrator.java`**: Matches incoming user queries to specialized skills, tools, and execution routing nodes.
* **`ScreenshotAnalyzer.java`**: Specialized multimodal agent component for processing user desktop screenshots and UI state.

### 2. Connectors & Data Integrations (`/connectors`)
Agents access external tools and context through `DataConnector` implementations managed by `ConnectorRegistry.java`:
* **Local & Workspace**: `AppleContactsConnector`, `AppleNotesConnector`, `IMessageConnector`, `ObsidianConnector`, `GranolaConnector`.
* **Cloud & Collaboration**: `GmailImapConnector`, `GDriveConnector`, `GCalendarConnector`, `GContactsConnector`, `SlackConnector`, `NotionConnector`, `DropboxConnector`, `WhatsAppConnector`.

### 3. Orchestration & Intelligence Routing (`/orchestration`)
* **`ComplexityClassifierNode.java`**: Classifies query complexity to determine whether tasks run via local lightweight LLMs or cloud models.
* **`ConditionalRouter.java`**: Dynamically routes tasks to appropriate execution nodes or tools based on context.
* **`OllamaModelClient.java` & `CloudModelClient.java`**: Uniform abstraction over local inference runtimes (e.g., Ollama) and cloud providers.

### 4. Memory & Private RAG (`/memory` & `/storage`)
* **`MemoryService.java` & `AgentMemory.java`**: Manages vector/text memory, local indexing, and context injection into agent prompts.
* **`IngestService.java`**: Ingests files and connected sources into local index repositories for search and long-term recall.

### 5. Workflows & Learning (`/workflow` & `/learning`)
* **`WorkflowService.java`**: Executes deterministic, multi-step pipeline tasks across agents.
* **`LearningService.java`**: Supports Spec Search optimization and agent self-improvement heuristics.

---

## Architectural Best Practices & Design Patterns

### 1. Strict Layer Isolation
* **Controllers (`/controllers`)**: Handle HTTP/WebSocket request validation, DTO mapping, and authentication. Controllers must never invoke repository persistence layers directly; all actions flow through domain services.
* **Services (`/services`, `/agents`)**: Contain all state transition logic, orchestration workflow execution, and domain validation.
* **Connectors (`/connectors`)**: Encapsulate third-party integration contracts and external APIs. Connectors strictly expose typed DTO responses and must handle external rate limits, exponential backoff, and circuit breaker patterns internally.
* **Repositories (`/storage`)**: Handle persistence and vector indexing. No agent execution decisions or LLM formatting logic are permitted inside data persistence layers.

### 2. Applied Design Patterns
* **Strategy Pattern**: Used in `SkillBasedOrchestrator` and model selection routing (`ConditionalRouter`). Custom routing heuristics or new tool resolvers must implement dedicated Strategy interfaces rather than sprawling `if-else` or untyped evaluation trees.
* **Factory Pattern**: Model client instances (`OllamaModelClient`, `CloudModelClient`) are created via `ModelClientFactory` based on node capabilities, budget thresholds, and user execution configs.
* **Observer Pattern**: State updates across long-running multi-agent tasks broadcast via event publishers (`AgentEventPublisher`) to push real-time status updates over WebSocket channels to the Tauri frontend.
* **Command Pattern**: Tool invocations (`ToolExecutionCommand`) are encapsulated as discrete executable objects, allowing deterministic execution logging, undo/rollback steps, and audit trails.

---

## Clean Code & Java 21 Runtime Guidelines

### 1. Project Loom & Virtual Thread Safety
* **Avoid Thread Pinning**: Never execute blocking I/O, network calls, or LLM HTTP requests inside a synchronized block or synchronized method. Use `ReentrantLock` instead of `synchronized` when synchronization across Virtual Threads is mandatory.
* **Structured Concurrency**: Use `StructuredTaskScope` (or Java 21 preview/virtual thread executors) when spawning parallel sub-agent calls in `MultiAgentCoordinator`. Ensure sub-tasks fail fast and cleanly propagate cancellations using `TaskScope.ShutdownOnFailure`.
* **Thread Locals**: Avoid using unbounded `ThreadLocal` storage for agent execution contexts, as millions of virtual threads can cause memory bloat. Use scoped values where possible or clear thread-local contexts explicitly.

### 2. Immutability & Modern Java Features
* **Java Records for DTOs & Events**: All event messages, tool payloads, and API requests must be declared using `record` types to ensure thread-safe immutability.
* **Sealed Interfaces for State & Errors**: Represent agent execution states (`AgentState`) and typed failure modes (`AgentError`) using `sealed interface` hierarchies to force exhaustive `switch` evaluation without fall-through bugs.
* **Null Safety**: Avoid returning `null` from service or connector calls. Return Java `Optional<T>` for potentially missing values, empty collections (`List.of()`), or explicit `Result<T, E>` types.

### 3. Error Handling & Security Best Practices
* **Structured Exception Handling**: Do not catch generic `Exception` or `Throwable` in core orchestrators. Catch specific operational exceptions (`LLMProviderTimeoutException`, `ToolExecutionException`) and map them to domain states.
* **Zero PII Leakage in Logs**: Never write raw prompt payloads containing API keys, user contact records, OAuth tokens, or unmasked message histories into application stdout or file logs. Mask credentials in connector trace outputs.
* **Graceful Degradation**: If a cloud model or external API connector fails or hits rate limits, the orchestrator must fall back to local Ollama execution or degrade tool availability safely rather than crashing active worker threads.

---

## Agent Persona Configurations

OpenTron provides pre-configured TOML specs in `configs/opentron/examples/`:

| Config File | Target Persona / Use Case | Tools Enabled |
| :--- | :--- | :--- |
| `chat-simple.toml` | Conversational lightweight assistant | None (Direct LLM response) |
| `code-assistant.toml` | Full-stack programming & shell agent | `code_interpreter`, `file_read`, `file_write`, `shell_exec`, `web_search`, `think`, `calculator` |
| `deep-research.toml` | Multi-hop document and memory research agent | `knowledge_search`, `knowledge_sql`, `scan_chunks`, `think`, `web_search` |
| `scheduled-monitor.toml` | Operative agent running scheduled tasks | `knowledge_search`, `knowledge_sql`, `memory_store`, `memory_search`, `think`, `web_search` |
| `morning-digest-mac.toml` | Personal morning voice summary agent | `digest_collect`, `text_to_speech`, `code_interpreter`, `web_search`, `file_read`, `shell_exec` |

---

## Development Workflows & Testing

### 1. Adding a New Agent Connector
1. Implement the `DataConnector` interface under `org.opentron.backend.connectors.impl`.
2. Encapsulate external data conversions into immutable DTO records.
3. Register the connector within `ConnectorRegistry.java`.
4. Expose state, OAuth, or configuration controllers in `ConnectorsController.java`.

### 2. Registering a Custom Agent Tool
1. Create a tool bean under `org.opentron.backend.tools` extending base tool behaviors.
2. Ensure execution logic is non-blocking or properly decoupled from virtual thread pinning locks.
3. Update `ToolsService.java` to handle tool call payloads.
4. Wire tool definitions into the frontend (`frontend/src/components/Chat/ToolCallCard.tsx`) for execution rendering.

### 3. Testing & Benchmarking Agents
Maintain strict unit test coverage for tool calls and orchestrator routing. Run native concurrency benchmarks to verify virtual thread behavior under load:

```bash
# Build backend executable with test verification
cd java/opentron-java/backend
mvn clean package

# Execute virtual thread concurrency benchmark suite
cd benchmarks
./benchmark.sh start
