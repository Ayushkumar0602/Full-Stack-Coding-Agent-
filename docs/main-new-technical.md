# Whizan Main Process — Technical Documentation

## Overview
`main-new.js` serves as the core **Main Process** entry point for the Whizan IDE. It orchestrates the lifecycle of the Electron application, manages the file system interface for the workspace, provides terminal emulation services, and acts as the backend bridge for AI-powered coding assistance via OpenRouter and Google Gemini APIs.

## Architecture
This file is the single source of truth for the Electron Main Process.
- **Upstream:** Interfaces directly with the Operating System (File System, Shell, Networking).
- **Downstream:** Communicates with the `renderer` process via `ipcMain`. All renderer-side actions (opening files, running terminal commands, prompting AI) are proxied through this module.
- **Dependencies:** Uses `node-pty` for low-level terminal integration, `fs-extra` for simplified filesystem operations, and `axios` for API communication.

## Design Principles
- **Separation of Concerns:** The module cleanly isolates platform-native logic (files, processes) from the UI.
- **IPC Gateway Pattern:** All interactions between the untrusted renderer and the system are funneled through defined `ipcMain.handle` endpoints, providing a secure boundary.
- **Resilient API Consumption:** Implements a "Key Rotation" pattern for AI providers. If a request fails due to rate-limiting (429) or unauthorized access (401), the module automatically switches to the next available API key in the configuration pool.
- **Process Management:** Employs a map-based registry (`terminalIdToPty`) to track and clean up child processes, preventing orphaned terminal instances.

## API Reference

### IPC Handlers (Renderer Interface)
| Channel | Input | Output | Description |
| :--- | :--- | :--- | :--- |
| `select-folder` | - | `string` | Opens native directory picker. |
| `read-file` | `filePath` | `{success, content}` | Reads text content from disk. |
| `write-file` | `filePath, content` | `{success}` | Writes/overwrites file at path. |
| `list-directory` | `dirPath` | `{files: Array}` | Recursively lists directory contents. |
| `create-terminal` | `id, shell` | `{success}` | Spawns a new `node-pty` instance. |
| `send-ai-message` | `message, context` | `{response}` | Routes prompt to selected AI model. |

## Internal Logic
1. **AI Initialization:** On startup, `initializeAI()` reads the static API key arrays. It maintains an index for key rotation per provider.
2. **Terminal Lifecycle:**
   - Terminals are tracked in `terminalIdToPty`.
   - Data events from `pty` are mirrored to the renderer via `mainWindow.webContents.send`.
   - `before-quit` event hook ensures all spawned shells are killed before process termination.
3. **API Routing:** The `send-ai-message` handler determines the provider (OpenRouter or Gemini) based on the `AI_MODELS` configuration object and dispatches to the corresponding helper function.

## Data Flow
1. **Request:** Renderer sends a payload via `ipcRenderer.invoke`.
2. **Processing:** Main process validates parameters, interacts with `fs` or `node-pty`, or performs an `axios` request.
3. **Response:** 
    - **Synchronous:** Results are returned via the `ipcMain` promise.
    - **Asynchronous (Terminal):** Data from the terminal process is pushed asynchronously via `mainWindow.webContents.send('terminal-data', ...)`.

## Error Handling & Edge Cases
- **API Failure:** Handled via a recursive retry mechanism. If an `axios` request fails with a 429/401 status, the code rotates the key and attempts a single retry.
- **File System:** All FS operations are wrapped in `try/catch` blocks and return a standard object format `{ success: boolean, error?: string }` to ensure the renderer can display graceful alerts.
- **Process Cleanup:** The `before-quit` handler prevents "zombie" processes by explicitly calling `kill()` on all active terminals.

## Usage Example

### Triggering an AI Request from Renderer
```javascript
// In the renderer process
const result = await window.electronAPI.invoke('send-ai-message', 'Refactor this function', 'currentFileContent...');

if (result.success) {
    displayResponse(result.response);
} else {
    console.error('AI Error:', result.error);
}
```

### Spawning a Terminal
```javascript
// In the renderer process
await window.electronAPI.invoke('create-terminal', 'terminal-1', 'zsh');

// Listen for output
window.electronAPI.on('terminal-data', (id, data) => {
    xtermInstance.write(data);
});
```