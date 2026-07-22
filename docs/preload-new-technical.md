# Preload new — Technical Documentation

## Overview
`preload-new.js` serves as the secure bridge between the Electron main process (Node.js environment) and the renderer process (browser environment). It acts as a gateway that restricts and defines the surface area for inter-process communication (IPC), ensuring that the frontend cannot directly access sensitive Node.js APIs while providing functional interfaces for file system manipulation, terminal emulation, and AI-driven development workflows.

## Architecture
This file is the **Preload Script** of an Electron application. 
- **Upstream:** It consumes `ipcRenderer` to dispatch requests to the Main process.
- **Downstream:** It exposes two primary objects (`electronAPI` and `api`) to the `window` object of the renderer process.
- **Dependency:** It relies on the Electron `contextBridge` module to maintain context isolation, a security best practice that prevents the renderer from interfering with the preload's internal logic.

## Design Principles
- **Context Isolation:** The module uses `contextBridge` to ensure the renderer process executes in an isolated environment, preventing prototype pollution or direct Node.js access.
- **Facade Pattern:** It provides simplified, task-oriented methods that abstract complex `ipcRenderer.invoke` calls, providing a cleaner API for frontend developers.
- **Backward Compatibility:** By maintaining both `electronAPI` and a legacy `api` namespace, the system supports phased migrations of frontend features without breaking existing modules.
- **Command-Query Separation:** While not strictly enforced, the design clearly separates event-based streams (listeners) from imperative commands (invocations).

## API Reference

### `window.electronAPI`
The modern interface for application-specific tasks.

*   **File System:** `selectFolder()`, `openFolder(path)`, `getWorkspacePath()`, `readFile(path)`, `writeFile(path, content)`, `createDirectory(path)`, `deleteFile(path)`, `renameFile(old, new)`, `getFileStats(path)`, `listDirectory(path)`.
*   **Terminal:** `createTerminal(id, shell)`, `writeTerminal(id, data)`, `resizeTerminal(id, cols, rows)`, `killTerminal(id)`.
*   **AI:** `sendAIMessage(msg, ctx)`, `setAIModel(name)`, `getAIModels()`, `getCurrentModel()`.
*   **Listeners:** Returns a cleanup function (unsubscriber) for `onTerminalData`, `onTerminalExit`, and `onWorkspaceLoaded`.

### `window.api`
The legacy interface encompassing extensive AI Agent, codebase indexing, and safety management features. 
*   **Key Extensions:** Includes comprehensive `aiAgent`, `aiGeneratePlan`, `codebaseSearch`, and multi-stage `aiUndo` operations.
*   **Safety APIs:** Provides `aiConfirmActions`, `aiEmergencyConfirm`, and `aiGetPendingActions` for human-in-the-loop oversight of AI file operations.

## Internal Logic
The module acts as a **proxy layer**. Every function follows a standard pattern:
1.  **Invocation:** A frontend action calls an `electronAPI` or `api` method.
2.  **IPC Dispatch:** The method invokes `ipcRenderer.invoke('channel-name', ...args)`.
3.  **Bridge:** The Electron Main process intercepts the channel name and executes the corresponding native Node.js logic (e.g., `fs.promises`, `node-pty`, or LLM API requests).
4.  **Event Subscription:** For real-time updates (e.g., terminal output or FS changes), the module wraps `ipcRenderer.on` listeners and returns a clean "off" function, preventing memory leaks by allowing the frontend to easily remove listeners.

## Data Flow
1.  **Requests:** Renderer → `contextBridge` → `ipcRenderer.invoke` → Main Process.
2.  **Responses:** Main Process → `ipcRenderer` (resolved Promise) → Renderer.
3.  **Events:** Main Process → `ipcRenderer.send` (events) → Preload Listener → Frontend Callback.

## Error Handling & Edge Cases
- **IPC Failures:** Since `ipcRenderer.invoke` returns a promise, errors in the main process (e.g., file not found, permission denied) are propagated as rejected promises to the renderer. Frontend code must use `try/catch` blocks.
- **Listener Management:** Each event-listener function returns a cleanup closure that calls `removeListener`. This is critical for single-page applications where components mount and unmount frequently.
- **Backward Compatibility:** The `api` namespace provides a promise-based `aiIsInitialized` to allow the frontend to verify the system readiness before firing commands.

## Usage Example

### Using the Modern API (Filesystem)
```javascript
async function loadFile(path) {
  try {
    const content = await window.electronAPI.readFile(path);
    console.log('File Content:', content);
  } catch (err) {
    console.error('Failed to read file:', err);
  }
}
```

### Using Event Listeners (Terminal)
```javascript
const unsubscribe = window.electronAPI.onTerminalData((terminalId, data) => {
  console.log(`Data from ${terminalId}:`, data);
});

// Clean up when the component is destroyed
// unsubscribe();
```