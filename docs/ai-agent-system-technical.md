# Ai agent system — Technical Documentation

## Overview
The `AIAgentSystem` is the central orchestration engine for the AI-driven development environment. Its primary purpose is to receive user natural language requests, transform them into actionable file system operations, and manage the lifecycle of these operations through analysis, contextual retrieval, AI-driven generation, safety validation, and persistent indexing.

## Architecture
This module sits at the heart of the agentic workflow. It acts as a controller that interfaces with several specialized managers:
- **`SessionManager`**: Handles long-term memory, incremental context, and operation history.
- **`SafetyManager`**: Manages file system integrity, transactional undo stacks, and permissions.
- **`CodebaseIndexer`**: Provides semantic awareness and dependency graphing of the local file system.

The `AIAgentSystem` is invoked by the main application process (usually via a GUI or CLI) and delegates heavy lifting to these dependencies, maintaining a centralized state of the current "workspace."

## Design Principles
- **Orchestration Pattern**: Uses a synchronous step-by-step pipeline (`processRequest`) to ensure predictable flow control.
- **Resilient Parsing**: Implements a "defensive programming" approach to LLM output, with multiple fallback strategies for JSON parsing to handle malformed or truncated responses.
- **Context-Aware Design**: Prioritizes incremental development by distinguishing between `new_request` and `follow_up` sessions, ensuring the AI maintains continuity.
- **Separation of Concerns**: Offloads safety and indexing logic to specialized classes (`SafetyManager`, `CodebaseIndexer`), keeping the main system focused on request orchestration.

## API Reference

### `new AIAgentSystem(workspacePath, codebaseIndexer)`
Initializes the system for a specific workspace.

### `processRequest(userRequest, currentFile, options)`
The main execution entry point.
- **`userRequest`** *(string)*: The goal defined by the user.
- **`currentFile`** *(object|null)*: Metadata for the active file context.
- **`options`** *(object)*: Config, including `aiService` implementation.
- **Returns**: A structured report containing status, operation ID, and execution summary.

### `undoLastAction()` / `undoMultipleActions(indices)`
Provides granular control over the `SafetyManager`'s transactional history for rolling back unintended file modifications.

### `getPendingActions()`
Retrieves actions currently held in the safety queue awaiting user approval.

## Internal Logic
1. **Analysis**: The request is broken down by intent (e.g., `create`, `refactor`) and complexity.
2. **Contextualization**: The `codebaseIndexer` performs semantic searches for relevant code snippets, filtered by `sessionContext` if it's a follow-up interaction.
3. **Prompt Engineering**: The `buildSystemPrompt` function constructs a comprehensive context, injecting project-specific coding standards, file structures, and dependency graphs.
4. **Generation & Parsing**: The system invokes the `aiService`. If the AI returns malformed JSON, the `parseAIResponse` logic executes a series of fixers (un-escaping, brace closing, and structural repair).
5. **Execution**: Validated actions are passed to `SafetyManager` for safe, atomic file operations.
6. **Persistence**: The results are indexed for subsequent queries, and the session state is updated.

## Data Flow
1. **User Request** $\to$ `processRequest`
2. **Analysis** $\to$ `retrieveContext` $\to$ **Relevant Code Chunks**
3. **Context + Request** $\to$ **AI Service** $\to$ **Raw Response**
4. **Raw Response** $\to$ **Parser** $\to$ **Action Queue**
5. **Action Queue** $\to$ `SafetyManager` $\to$ **Disk Operations**
6. **Result** $\to$ **Session Update** $\to$ **User Feedback**

## Error Handling & Edge Cases
- **JSON Failure**: If AI responses fail to parse, the system returns a fallback object instead of crashing, ensuring the UI can still communicate failure to the user.
- **Truncated Responses**: `parseLargeJsonResponse` attempts to detect incomplete JSON and dynamically injects missing closing tags.
- **AI Timeout**: All network calls to the AI service are wrapped in a 120-second `Promise.race` timeout to prevent process hangs.
- **Atomic Operations**: By leveraging the `SafetyManager`, the system supports undo operations, effectively acting as a "transactional filesystem" layer.

## Usage Example

```javascript
const agent = new AIAgentSystem(myWorkspacePath, myIndexer);

// Processing a standard request
const response = await agent.processRequest(
  "Create a new login component with validation", 
  activeFilePath, 
  { aiService: myLLMConnector }
);

if (response.success) {
  console.log(`Action completed: ${response.operationId}`);
} else {
  console.error(`Error: ${response.error}`);
}
```

```javascript
// Rolling back accidental changes
const undoStatus = await agent.undoLastAction();
console.log('Rollback status:', undoStatus.success);
```