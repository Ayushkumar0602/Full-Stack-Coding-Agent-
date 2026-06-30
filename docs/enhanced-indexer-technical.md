# Enhanced indexer — Technical Documentation

## Overview
The `EnhancedCodebaseIndexer` is a robust, production-grade utility designed to transform a local filesystem into a machine-readable, semantically aware knowledge graph. It is built to serve as the foundational indexing layer for LLM-powered coding assistants or automated documentation tools, providing deep insights into code structure, intent, documentation, and cross-file dependencies.

## Architecture
This module acts as the primary data ingestion engine. It traverses the workspace, tokenizes code into semantically meaningful chunks, and constructs a multi-layered index.

*   **Upstream:** Consumes raw file streams from the disk via `fs-extra`.
*   **Downstream:** The generated index (accessible via the `index` property or helper methods) serves as a data provider for semantic search, context window management, and repository analysis tools.
*   **Dependency:** Depends on `fs-extra` for file system operations, `path` for cross-platform directory resolution, and `crypto` for integrity verification.

## Design Principles
*   **Semantic Chunking:** Rather than splitting code by arbitrary line counts, the indexer uses regex-based "semantic boundaries" (e.g., function definitions, import blocks, JSDoc). This ensures that each chunk maintains high context integrity.
*   **Intent-Based Categorization:** Code is classified by architectural intent (e.g., `authentication`, `businessLogic`), allowing for filtered retrieval.
*   **Incremental Updates:** The system uses MD5 hashing per file to detect changes, enabling efficient, partial re-indexing of modified files rather than performing full scans.
*   **Separation of Concerns:** Logic is divided into distinct phases: scanning, documentation extraction, dependency mapping, and semantic vector generation.

## API Reference

### `new EnhancedCodebaseIndexer(workspacePath)`
*   **`workspacePath`** (string): The absolute path to the directory to be indexed.

### `async indexCodebase()`
Performs a full scan, parses files, builds the graph, and generates embeddings. This is the primary entry point for the indexing lifecycle.

### `async searchSemantic(query, options)`
Executes a multi-factor search over the indexed chunks.
*   **`query`** (string): The search terms.
*   **`options.limit`** (number): The maximum number of results to return.
*   **Returns:** An array of objects containing `chunkId`, `score`, `snippet`, and `matchType`.

### `getFileInfo(filePath)`
Returns a comprehensive metadata object for a specific file, including its dependency graph, extracted documentation, and pre-calculated context.

### `updateIndexForChangedFiles(changedFiles)`
Updates the index for a subset of files. Used to keep the index in sync with IDE file-watchers.

## Internal Logic
1.  **Scanning:** Performs a recursive walk, filtering based on `ignorePatterns`.
2.  **Chunking:** The `generateSemanticChunks` method iterates through lines, checking `isSemanticBoundary` to define split points.
3.  **Metadata Extraction:** Extracts JSDoc, parameter types, return signatures, and inline comments using specific regex patterns.
4.  **Vectorization:** The `createSemanticVector` method generates a weighted frequency map of identifiers, purpose, and intent tags to facilitate semantic matching.
5.  **Graph Construction:** The `buildEnhancedDependencyGraph` scans file content for imports/exports to determine inter-file relationships.

## Data Flow
1.  **Input:** `workspacePath` (Disk files).
2.  **Transformation:** 
    *   Files are filtered and stats gathered.
    *   Content is hashed and compared to detect changes.
    *   Files are partitioned into chunks and labeled with metadata (intent, complexity, language).
3.  **Output:** An internal `index` object store, containing maps for `files`, `chunks`, `embeddings`, and a `dependencyGraph`.

## Error Handling & Edge Cases
*   **Graceful Degradation:** File system errors (permission denied, locked files) are caught and logged with a warning, allowing the indexing of the remaining tree to continue.
*   **Cycle Detection:** Circular imports are managed via the `dependencyGraph` structure, which only records explicit matches found during the current sweep.
*   **Parsing Failures:** Regex-based extraction includes `try-catch` blocks and safety checks to prevent crashes on malformed files or unexpected file contents.

## Usage Example

### Basic Indexing
```javascript
const indexer = new EnhancedCodebaseIndexer('./my-project');
await indexer.indexCodebase();

const results = await indexer.searchSemantic('user authentication', { limit: 5 });
console.log(results[0].snippet);
```

### Partial Update (e.g., via File Watcher)
```javascript
// Triggered by a file system watcher
const changedFiles = ['src/auth/login.ts'];
await indexer.updateIndex(changedFiles);
```