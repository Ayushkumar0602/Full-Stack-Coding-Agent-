# Main — Technical Documentation

## Overview
`main.js` serves as the core entry point and backend process orchestrator for the **Whizan** application. Built on Electron, it acts as the central hub for managing the application lifecycle, IPC (Inter-Process Communication) bridge between the UI and native system resources, AI agent coordination, terminal emulation, and filesystem operations.

## Architecture
This file resides at the root of the Electron main process. It operates as the "Source of Truth" for the application state (e.g., `workspacePath`, `isBuildingProject`, and active AI providers).

*   **Upstream:** Consumes `codebase-indexer.js`, `enhanced-indexer.js`, `ai-agent-system.js`, and `session-manager.js` to perform heavy-lifting tasks.
*   **Downstream:** Serves data via `ipcMain` channels to the renderer process (`preload.js` bridge).
*   **External Dependencies:** Leverages `node-pty` for terminal emulation, `chokidar` for filesystem observation, and `axios` for LLM communication.

## Design Principles
*   **Separation of Concerns:** The module delegates domain-specific logic (indexing, agent reasoning) to imported classes, keeping `main.js` focused on orchestration.
*   **Resiliency:** Implements multi-key fallback strategies for AI providers to handle rate limits (429 errors) and service outages gracefully.
*   **State Management:** Utilizes a singleton-like pattern for the workspace and AI system, ensuring that application-wide changes (like changing a workspace folder) trigger global updates.
*   **Asynchronous Processing:** Nearly all I/O, AI calls, and shell executions are promisified to ensure the main process does not block the UI.

## API Reference

### AI Core
*   `callAI(messages, options, onProgress)`: Routes requests to the active AI provider (Gemini or OpenRouter) with automatic fallback.
*   `generateProjectPlan(prompt, onProgress)`: Generates structured JSON describing project architecture and initialization steps.
*   `generateCode(filePath, context, requirements, onProgress)`: Generates production-ready file content based on provided prompt context.

### Lifecycle & File System
*   `startWatcher(rootPath)`: Initializes a `chokidar` instance to monitor changes in the user workspace, triggering `codebaseIndexer` updates.
*   `createProjectStructure(structure)`: Orchestrates directory and file creation using `fs-extra` to mirror the AI-generated project plan.
*   `executeCommand(command, cwd)`: Spawns sub-processes for system-level operations (e.g., `npm install`).

### IPC Handlers (`setupIpc`)
*   `pty:create`: Spawns a `node-pty` pseudo-terminal instance.
*   `ai:generatePlan`: Entry point for UI-driven project generation.
*   `ai:buildProjectWithStreaming`: Orchestrates the automated setup of project scaffolding, dependencies, and file generation.

## Internal Logic
1.  **AI Orchestration:** The module initializes with hardcoded API key arrays for both Gemini and OpenRouter. It tracks a `currentKeyIndex` for both, incrementing on failure (round-robin style) to bypass rate limits.
2.  **Streaming:** The `handleOpenRouterStream` function parses raw server-sent events, buffering and accumulating code chunks before broadcasting progress and updates to the renderer via `onProgress` callbacks.
3.  **Workspace Lifecycle:** When a folder is selected via `dialog`, the `loadEditor` flow initiates:
    *   `workspacePath` is updated.
    *   `CodebaseIndexer` creates a semantic map of the codebase.
    *   `AIAgentSystem` is instantiated with the new index.
    *   UI navigates to the editor view.

## Data Flow
1.  **Input:** User interaction in the renderer process triggers `ipcMain.handle` via `contextBridge`.
2.  **Processing:** `main.js` validates inputs and delegates logic to AI services or File System utilities.
3.  **Feedback:** During long-running tasks (e.g., `buildProjectWithStreaming`), the module pushes updates to the renderer via `event.sender.send` (e.g., `ai:buildProgress`, `ai:codeStream`).
4.  **Output:** Final data (paths, status objects) is returned to the original IPC invoker as a `Promise` result.

## Error Handling & Edge Cases
*   **AI Failover:** If Gemini returns a 503 or 400 error, `callAI` automatically attempts to switch to the OpenRouter/DeepSeek configuration.
*   **Zombie Terminals:** `terminalIdToPty` Map tracks active terminal processes; if a window closes or a process terminates, it is removed from the Map and destroyed.
*   **File Conflicts:** Filesystem operations use `fs-extra` to ensure existence (`ensureFile`, `ensureDir`) before attempting writes, preventing crashes on missing paths.
*   **Malformed AI Responses:** JSON parsers in `generateProjectPlan` include fallback regex logic to extract JSON objects from conversational text blocks.

## Usage Example

### Triggering a Project Build
```javascript
// Consumed from the renderer to start a fully automated build
const result = await window.electronAPI.invoke('ai:buildProjectWithStreaming', {
  setupCommand: "npx create-react-app my-app",
  structure: { /* ... */ },
  dependencies: { frontend: ["tailwind"] }
});
```

### Accessing the Terminal
```javascript
// Creating a terminal for a user session
const terminal = await window.electronAPI.invoke('pty:create', 80, 24);
// Writing a command to that terminal
await window.electronAPI.invoke('pty:write', terminal.id, 'npm start\n');
```