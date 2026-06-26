# Codebase indexer — Technical Documentation

## Overview
The `CodebaseIndexer` is a specialized utility designed to traverse a local file system, extract semantic and structural information from source code, and maintain an in-memory index for rapid retrieval. It acts as the backbone for RAG (Retrieval-Augmented Generation) pipelines by enabling semantic search, dependency mapping, and context-aware retrieval across diverse programming languages.

## Architecture
This module functions as an autonomous service within a larger codebase intelligence system. 
- **Upstream:** It consumes raw files from the local disk.
- **Internal:** It maintains state through several `Map` objects, categorizing data into files, chunks, dependencies, and vector representations.
- **Downstream:** It provides an API for higher-level agents (like LLMs or IDE extensions) to perform semantic queries and fetch code context.

## Design Principles
- **State Preservation:** Uses MD5 hashing to compare file contents before processing, ensuring incremental updates are efficient and avoiding unnecessary re-indexing.
- **Modular Scanning:** Employs a recursive file-tree traversal with customizable ignore patterns to keep the index noise-free.
- **Language Agnostic (Extensible):** Logic is decoupled from language-specific regex patterns, allowing for easy expansion by updating the `pattern` dictionaries.
- **Single Responsibility:** Separates concern between file discovery, structural parsing (dependency graph), and vectorization (embeddings).

## API Reference

### `new CodebaseIndexer(workspacePath: string)`
Initializes the indexer with a root directory.

### `async indexCodebase(): Promise<void>`
Full recursive scan of the `workspacePath`. Orchestrates the indexing lifecycle.

### `async semanticSearch(query: string, maxResults?: number): Promise<Array>`
Performs a cosine similarity search against indexed chunks. Returns a list of chunks ranked by relevance.

### `async getRelevantContext(query: string, maxChunks?: number): Promise<Object>`
A high-level utility that wraps `semanticSearch` to produce a structured context object for external LLM consumption.

### `async updateIndex(changedFiles: string[]): Promise<void>`
Triggers an incremental update on specific files, re-parsing only the paths provided.

### `getIndexStats(): Object`
Returns a summary report of the current memory index (counts, language distribution).

## Internal Logic
1.  **Scanning:** Performs a depth-first traversal of the filesystem, filtering by `ignorePatterns` and `indexableExtensions`.
2.  **Processing:** 
    *   Generates an MD5 hash of the file content.
    *   Skips if the hash matches the cached version.
    *   Splits files into semantic "chunks" of ~500 characters while preserving line integrity.
3.  **Parsing:** `extractDependencies` scans lines for language-specific keywords (imports/exports/functions/classes) using a pre-defined regex map.
4.  **Vectorization:** `createSimpleEmbedding` builds a frequency map of terms to represent chunks, allowing for basic vector arithmetic (cosine similarity).

## Data Flow
1.  **Input:** Filesystem path -> `scanFileTree`.
2.  **Transformation:** File content -> `generateChunks` (Segmentation) -> `createSimpleEmbedding` (Vectorization).
3.  **Storage:** Maps are updated to store file metadata, chunk locations, and dependency trees.
4.  **Output:** `semanticSearch` yields objects containing `chunkId`, `similarity` scores, and original source metadata.

## Error Handling & Edge Cases
- **Permission Errors:** `scanFileTree` utilizes `try/catch` blocks at the directory level to prevent one unreadable directory from crashing the entire index.
- **Large Files/Binary Files:** Handled by the `isIndexable` whitelist and `ignorePatterns`.
- **Duplicate Logic:** The `fileHashes` map prevents redundant heavy processing for files that haven't changed.
- **Missing Dependencies:** `extractDependencies` uses regex arrays; if a language is unsupported, it defaults to empty dependency sets rather than throwing exceptions.

## Usage Example

### Initializing and Indexing
```javascript
const indexer = new CodebaseIndexer('./my-project');
await indexer.indexCodebase();
console.log(indexer.getIndexStats());
```

### Retrieving Context for a Query
```javascript
const context = await indexer.getRelevantContext('How does the authentication middleware work?', 3);
// Returns an object containing the top 3 relevant code chunks and their source metadata
console.log(context.relevantChunks);
```

### Incremental Update
```javascript
// Triggered when a file watcher detects a change
await indexer.updateIndex(['src/auth.js', 'src/db.js']);
```