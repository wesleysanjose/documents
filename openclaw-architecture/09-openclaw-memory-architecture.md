# OpenClaw Memory & Knowledge System Architecture

> Deep analysis of how OpenClaw designs its memory, knowledge retrieval, and context management systems, and how they build on Pi Agent's session primitives.

---

## 1. Architecture Overview

OpenClaw's memory system is a **three-layer architecture** that transforms user-authored markdown files and session transcripts into a searchable semantic index, making long-term knowledge available to agents across sessions.

```
Layer 3: Agent Interface
  memory_search tool, memory_get tool, bootstrap injection

Layer 2: Memory Index (OpenClaw)
  SQLite store, hybrid search, embedding pipeline, sync engine

Layer 1: Session Primitives (Pi Agent)
  SessionManager, JSONL storage, compaction, branching
```

**Key design principle**: Workspace markdown files (MEMORY.md, memory/*.md) are the **source of truth**. The SQLite index is a derived, rebuildable cache. If the index is lost or corrupted, it can be fully regenerated from the source files.

---

## 2. Storage Architecture

### 2.1 Source of Truth: Workspace Files

Agents read and write memory as plain markdown files in the workspace:

| File | Purpose |
|------|---------|
| `MEMORY.md` | Primary memory file, always loaded into system prompt |
| `memory/*.md` | Topic-specific memory files (e.g., `memory/debugging.md`, `memory/2025-03-01.md`) |
| Extra paths | User-configured additional directories to index |

This design is intentional: files are human-readable, version-controllable, and portable. The agent literally reads and writes the same files a human would.

### 2.2 SQLite Index Database

Located at `~/.openclaw/state/memory/{agentId}.sqlite`, the index contains 6 tables:

**`meta`** - Key-value metadata tracking index state:
```sql
CREATE TABLE meta (
  key TEXT PRIMARY KEY,    -- model, provider, providerKey, chunkTokens, chunkOverlap, vectorDims
  value TEXT NOT NULL
);
```

**`files`** - File tracking for incremental sync:
```sql
CREATE TABLE files (
  path TEXT PRIMARY KEY,
  source TEXT NOT NULL DEFAULT 'memory',  -- 'memory' | 'sessions'
  hash TEXT NOT NULL,
  mtime INTEGER NOT NULL,
  size INTEGER NOT NULL
);
```

**`chunks`** - Core content storage with embeddings:
```sql
CREATE TABLE chunks (
  id TEXT PRIMARY KEY,          -- deterministic hash
  path TEXT NOT NULL,
  source TEXT NOT NULL DEFAULT 'memory',
  start_line INTEGER NOT NULL,  -- line tracking for citations
  end_line INTEGER NOT NULL,
  hash TEXT NOT NULL,
  model TEXT NOT NULL,           -- embedding model used
  text TEXT NOT NULL,            -- original chunk text
  embedding TEXT NOT NULL,       -- Float32Array as base64
  updated_at INTEGER NOT NULL
);
```

**`embedding_cache`** - Provider-specific cache to avoid re-embedding:
```sql
CREATE TABLE embedding_cache (
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  provider_key TEXT NOT NULL,
  hash TEXT NOT NULL,
  embedding TEXT NOT NULL,
  dims INTEGER,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (provider, model, provider_key, hash)
);
```

**`chunks_fts`** - FTS5 virtual table for keyword search:
```sql
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  text,
  id UNINDEXED, path UNINDEXED, source UNINDEXED,
  model UNINDEXED, start_line UNINDEXED, end_line UNINDEXED
);
```

**`chunks_vec`** - sqlite-vec virtual table for vector search:
```sql
-- Created dynamically when sqlite-vec extension loads
-- Stores Float32 embedding vectors for cosine similarity
```

### 2.3 Why SQLite?

- **Single-file portability**: One `.sqlite` file per agent, easy to backup/migrate
- **Atomic reindexing**: Temp DB + atomic swap prevents corruption during reindex
- **Native extensions**: FTS5 for keyword search, sqlite-vec for vector search
- **Zero infrastructure**: No external database server needed
- **Graceful degradation**: If sqlite-vec fails to load, falls back to FTS-only mode

---

## 3. Embedding Pipeline

### 3.1 Providers

OpenClaw supports 4 embedding providers with automatic selection:

| Provider | Default Model | Dimensions | Max Tokens | Batch API |
|----------|--------------|------------|------------|-----------|
| **OpenAI** | `text-embedding-3-small` | 1536 | 8192 | Yes (async file upload) |
| **Gemini** | `gemini-embedding-001` | varies | varies | Yes |
| **Voyage** | `voyage-4-large` | varies | varies | Yes |
| **Local** | `embeddinggemma-300m` (GGUF) | varies | 2048 | No |

**Auto-selection order**: local (if model file exists) -> openai -> gemini -> voyage

**Fallback**: If the primary provider fails, the system can fall back to a configured alternative (e.g., openai -> gemini).

### 3.2 Chunking Strategy

File: `src/memory/internal.ts` (`chunkMarkdown()`)

- **Token limit**: 400 tokens per chunk (default)
- **Overlap**: 80 tokens between adjacent chunks
- **Character estimation**: `maxChars = max(32, tokens * 4)`, `overlapChars = overlap * 4`
- **Line-based splitting**: Chunks track `start_line` and `end_line` for precise citations
- **Minimum chunk size**: 32 characters

The chunking is line-aware: it splits on line boundaries and tracks exact line numbers, enabling the agent to cite `memory/debugging.md:15-28` in its responses.

### 3.3 Embedding Processing Pipeline

```
File changed detected
  |
  v
chunkMarkdown(content, {tokens: 400, overlap: 80})
  |
  v
Hash each chunk (deterministic ID)
  |
  v
Check embedding_cache for existing embeddings
  |--- Cache hit: reuse cached embedding
  |--- Cache miss: request from provider
  |
  v
embedBatchWithRetry(chunks, {retries: 3, backoff: 500ms-8s})
  |
  v
sanitizeEmbedding() - replace NaN/Inf with 0
  |
  v
L2 normalize -> Float32Array
  |
  v
Store in chunks table + FTS index + vector table
  |
  v
Update embedding_cache
```

### 3.4 Batch Processing

For large reindexing operations, OpenClaw supports async batch APIs:
- **OpenAI Batch API**: Upload JSONL file, poll for completion (cost reduction)
- **Gemini Batch API**: Similar async job model
- **Voyage Batch API**: Similar async job model

Batch processing is useful during full reindex (hundreds of chunks), while individual embedding requests are used for incremental updates.

### 3.5 Security: Guarded Remote HTTP

File: `src/memory/remote-http.ts`

All remote embedding API calls (and batch API calls) are routed through a **guarded remote HTTP helper** that enforces SSRF protection:
- Builds an allow-list policy from the configured base URL (`allowedHostnames: [parsed.hostname]`)
- Cross-host redirects are blocked while configured operator endpoints continue to work
- Uses `fetchWithSsrFGuard()` from `src/infra/net/fetch-guard.ts`
- Applied to all embedding providers and batch APIs

### 3.6 Input Hard-Caps

Embedding inputs are hard-capped before being sent to any provider:
- **Local provider**: 2048 tokens max (added to prevent OOM with small GGUF models)
- **Gemini**: 2048 tokens max
- **OpenAI**: 8192 tokens max
- Chunks exceeding the cap are truncated before embedding, preventing batch failures

---

## 4. Hybrid Search Architecture

### 4.1 Search Pipeline

File: `src/memory/hybrid.ts`, `src/memory/manager-search.ts`

```
User query: "What did we decide about the API authentication?"
  |
  v
Query Expansion (query-expansion.ts)
  - Remove stop words (English, Chinese, Japanese, Korean, Spanish, Portuguese, Arabic)
  - Tokenize CJK characters (unigrams + bigrams)
  - Validate keyword length (min 3 chars for English)
  - Result: ["decided", "API", "authentication"]
  |
  v
Parallel execution:
  |
  |---> Vector Search (70% weight)
  |     embed(query) -> cosine similarity via sqlite-vec
  |     vec_distance_cosine(embedding, query_vec)
  |     Score: 1 - cosine_distance
  |
  |---> Keyword Search (30% weight)
  |     FTS5 query: "decided" AND "API" AND "authentication"
  |     BM25 ranking: 1 / (1 + rank)
  |
  v
Merge by chunk ID (hybrid.ts::mergeHybridResults)
  combined_score = 0.7 * vector_score + 0.3 * text_score
  |
  v
Optional: Temporal Decay (temporal-decay.ts)
  - Extract date from file path: memory/YYYY-MM-DD.md
  - Root files (MEMORY.md) are "evergreen" (no decay)
  - Formula: score * exp(-lambda * age_days)
  - Default half-life: 30 days
  |
  v
Optional: MMR Re-ranking (mmr.ts)
  - Maximal Marginal Relevance for diversity
  - mmr_score = lambda * relevance - (1-lambda) * max_similarity_to_selected
  - Jaccard similarity between snippets
  - Default lambda: 0.7
  |
  v
Filter: minScore >= 0.35, maxResults = 6
  |
  v
Return: [{path, startLine, endLine, score, snippet, source}]
```

### 4.2 Why Hybrid Search?

| Approach | Strengths | Weaknesses |
|----------|-----------|------------|
| Vector-only | Semantic understanding, handles paraphrasing | Misses exact terms, costs API calls |
| Keyword-only | Exact matches, zero latency, no API cost | No semantic understanding |
| **Hybrid (70/30)** | Best of both: semantic + exact matching | Slightly more complex |

The 70/30 split was chosen because semantic understanding is more important for memory recall (you rarely remember the exact words you used), but keyword matching catches precise terms like tool names, project names, and dates.

### 4.3 Graceful Degradation

```
Best case:  sqlite-vec + FTS5 + embeddings -> full hybrid search
Fallback 1: Keyword hits preserved when vector search returns empty (hybrid keeps FTS results)
Fallback 2: FTS5 only (no vector extension) -> keyword search only
Fallback 3: In-memory cosine similarity (sqlite-vec unavailable, embeddings exist)
Fallback 4: QMD external sidecar (alternative backend)
```

**Important fix**: When the vector search returns no results but keyword search does, the hybrid merger now preserves the keyword hits rather than discarding them. This ensures exact-match queries always surface results even when embeddings miss.

---

## 5. Memory Synchronization Engine

### 5.1 Sync Triggers

The index stays up-to-date through multiple trigger mechanisms:

| Trigger | When | Scope |
|---------|------|-------|
| **File watch** | MEMORY.md or memory/*.md changes | Memory files only, debounced 1500ms |
| **On search** | Before every search query | Both memory + sessions (if dirty) |
| **On session start** | Agent session begins | Memory files only |
| **Interval** | Configurable periodic sync | Both memory + sessions |
| **Session delta** | Session JSONL grows by 100KB or 50 messages | Session files only |
| **Source change** | Memory `sources` config changes (e.g., adding "sessions") | Full reindex triggered |

### 5.2 Incremental vs Full Reindex

**Incremental sync** (default): Only re-embed files whose hash has changed.
```
For each file:
  DB record = SELECT hash FROM files WHERE path = ?
  IF record.hash == currentFile.hash:
    skip (no change)
  ELSE:
    re-chunk and re-embed
```

**Full reindex** (on config change, model change, or manual trigger):
```
1. Create temp database: {dbPath}.tmp-{uuid}
2. Seed embedding cache from original DB (reuse cached embeddings)
3. Index ALL memory files into temp DB
4. Index ALL session files into temp DB
5. Write new metadata
6. Close both databases
7. Atomic swap: rename temp -> original (with WAL/SHM handling)
8. Reopen database
9. On failure: rollback (restore original, delete temp)
```

This atomic swap ensures the index is never in a half-written state, even if the process crashes during reindex.

**Readonly filesystem recovery**: If the index database is on a readonly filesystem, the sync engine detects this gracefully and operates in read-only mode without crashing. It will retry when the filesystem becomes writable again.

**Async sync close race fix**: A race condition where `close()` could be called while an async sync was in progress has been fixed, preventing database corruption during shutdown.

### 5.3 File Watcher

Uses chokidar to watch workspace files:
- Watches: `MEMORY.md`, `memory/**/*.md`, configured extra paths
- Ignores: `.git`, `node_modules`, `.pnpm-store`, `.venv`, `__pycache__`
- Debounce: 1500ms (configurable)
- On change: marks `dirty = true`, schedules sync

---

## 6. Memory Flush: Pre-Compaction Persistence

### 6.1 The Problem

When Pi's context window fills up, compaction summarizes old messages and discards them. But valuable information in those messages might not yet be saved to memory files.

### 6.2 The Solution: Memory Flush

File: `src/auto-reply/reply/memory-flush.ts`

Before compaction runs, OpenClaw injects a special "memory flush" turn:

```
System: "Pre-compaction memory flush turn. The session is near auto-compaction;
         capture durable memories to disk."

User:   "Pre-compaction memory flush. Store durable memories now
         (use memory/YYYY-MM-DD.md; create memory/ if needed).
         IMPORTANT: If the file already exists, APPEND new content only
         and do not overwrite existing entries.
         If nothing to store, reply with <silent>."
```

The agent then:
1. Reviews the conversation about to be compacted
2. Writes any important facts/decisions/preferences to `memory/YYYY-MM-DD.md`
3. Replies with `<silent>` if nothing to save

**Trigger condition**: Session tokens approach the soft threshold (default 4000 tokens before compaction limit).

**Token accounting fix**: The flush gating logic now correctly accounts for context tokens when deciding whether to trigger, preventing premature or missed flushes. Byte-size parsing for the soft threshold has been extracted to a shared `src/config/byte-size.ts` helper.

### 6.3 Why This Matters

This is a critical design decision: it means **no information is lost during compaction**. The agent gets a chance to persist anything important before it's summarized away. The date-stamped filenames also create a natural timeline of what the agent learned and when.

---

## 7. Context Management (OpenClaw Extensions to Pi)

### 7.1 Pi's Session Primitives

Pi Agent provides the foundational session management:

**SessionManager** (`@mariozechner/pi-agent-core`):
- **Storage**: Append-only JSONL files at `~/.openclaw/agents/{agentId}/sessions/*.jsonl`
- **Branching**: Tree structure via `parentUuid` links (sessions can branch)
- **Compaction**: LLM-driven summarization when context exceeds limits
  - `findCutPoint()`: Walk backwards from newest, accumulate tokens, find clean cut
  - `generateSummary()`: LLM summarizes discarded messages
  - `compact()`: Replace old messages with summary + kept recent messages
  - Tracks `FileOperations` (files created/modified/deleted) across compacted messages
- **Entries**: Messages, tool calls, tool results, bash outputs, compaction summaries

**CompactionSettings** (`settings.jsonl`):
```typescript
{
  keepRecentTokens: number;    // How many tokens to keep after cut
  reserveTokens: number;       // Reserve for summary generation
  customInstructions?: string; // Focus areas for summary
}
```

### 7.2 OpenClaw's Extensions

OpenClaw wraps Pi's primitives with additional capabilities:

**Session Write Lock** (`src/agents/session-write-lock.ts`):
- File-based locking (`.lock` files with PID + process start time)
- Prevents concurrent session writes from multiple agents
- Stale lock detection: 30min default, checks PID alive
- **PID recycling detection** (Linux): Records process start time (`/proc/pid/stat` field 22) in lock file; if PID is alive but start time differs, the lock is treated as stale and reclaimed (prevents container PID reuse from blocking locks)
- **Orphan self-PID lock reclamation**: Detects and reclaims lock files left by the current process from a previous crash
- **Lock contention hardening**: Improved fairness under concurrent access with FIFO queue
- Watchdog: 60s interval checking for expired locks
- Graceful cleanup on SIGINT/SIGTERM/SIGQUIT/SIGABRT

**Context Pruning** (`src/agents/pi-extensions/context-pruning/`):
- A Pi extension that removes individual messages mid-conversation
- Uses `cache-ttl` mode: prunes messages based on cache touch time
- Configurable per-tool prunability (`isToolPrunable`)
- More surgical than compaction: removes specific messages instead of summarizing a range

**Session Manager Runtime Registry**:
- WeakMap-based registry keyed by SessionManager instance
- Attaches OpenClaw runtime metadata to Pi sessions
- Used by context pruning, memory flush, and other extensions

**Bootstrap Injection**:
- On session start, memory search results are injected into the system prompt
- MEMORY.md content is always loaded (up to 20KB per file, 150KB total)
- Additional context from memory/*.md files included based on relevance

### 7.3 Tool Result Truncation

Large tool results (file reads, API responses) are truncated to prevent context overflow:
- **Max size**: 400,000 characters
- Applied transparently before the result enters the context
- Preserves the beginning and end of large outputs

### 7.4 History Limiting

Configurable limit on how many messages are kept in the active context:
- Prevents unbounded growth even without compaction
- Works alongside compaction as a safety net

---

## 8. QMD Backend (Alternative Search Sidecar)

File: `src/memory/qmd-manager.ts`, `src/memory/qmd-query-parser.ts`

QMD is an external search tool that can replace or supplement the built-in SQLite search:

```
Architecture:
  OpenClaw Agent -> FallbackMemoryManager
    |-> Try QMD primary (external process)
    |-> Fall back to SQLite index on error
```

**QMD Features**:
- BM25 + vector search + reranking
- Better performance on large knowledge bases
- Separate update/embed intervals (5min update, 60min embed)
- Manages collections: memory paths, custom paths, session exports
- **Han-script BM25 normalization**: CJK queries are preprocessed through `extractKeywords()` to filter single-character unigrams (too broad for BM25), deduplicate, and cap at 12 keywords
- **Mixed-source result diversification**: When results come from both memory and session sources, QMD interleaves them to avoid one source dominating
- **Legacy collection migration**: Automatically migrates unscoped collections to the new scoped format
- **mcporter keep-alive**: QMD searches can be routed through mcporter for persistent connections
- **stdout discarding**: Update/embed operations discard stdout to prevent output cap failures on large knowledge bases

**Query Modes**:
- `search`: Faster, lower recall (Han-script normalization applied)
- `query`: Slower, better recall (Han-script normalization applied)

**Fallback Wrapper** (`search-manager.ts::FallbackMemoryManager`):
- Tries QMD first
- Falls back to built-in SQLite on any error
- Transparent to the calling code

---

## 9. Agent Interface

### 9.1 Memory Search Tool

File: `src/agents/tools/memory-tool.ts`

```typescript
{
  name: "memory_search",
  description: "Mandatory recall step: semantically search MEMORY.md + memory/*.md
    (and optional session transcripts) before answering questions about prior work,
    decisions, dates, people, preferences, or todos",
  parameters: {
    query: string,        // Search query
    maxResults?: number,  // Override default 6
    minScore?: number     // Override default 0.35
  }
}
```

The tool description says "Mandatory recall step" -- this is intentional. It trains the agent to always check memory before answering from (potentially outdated) context.

**Unavailable status**: When memory search is disabled or the backend is unreachable, the tool now returns an explicit `{ results: [], disabled: true, error: "..." }` response rather than silently failing, giving the agent clear feedback about why recall is unavailable.

### 9.2 Memory Get Tool

```typescript
{
  name: "memory_get",
  parameters: {
    path: string,     // File path within workspace
    from?: number,    // Starting line
    lines?: number    // Number of lines to read
  }
}
```

Direct file access for when the agent knows exactly which memory file it needs.

### 9.3 Citations

Memory search results include citation metadata:
- `path`: File path (e.g., `memory/debugging.md`)
- `startLine` / `endLine`: Exact line numbers
- `score`: Relevance score
- `source`: Whether from memory files or session transcripts

Citations can be enabled/disabled based on the communication channel (enabled for CLI/web, disabled for messaging channels like Slack/Telegram).

---

## 10. Configuration

File: `src/agents/memory-search.ts`

```typescript
ResolvedMemorySearchConfig {
  enabled: true,
  sources: ["memory"] | ["memory", "sessions"],
  provider: "auto" | "openai" | "local" | "gemini" | "voyage",
  fallback: "openai" | "gemini" | "local" | "voyage" | "none",
  model: "text-embedding-3-small",
  chunking: { tokens: 400, overlap: 80 },
  sync: {
    onSessionStart: true,
    onSearch: true,
    watch: true,
    watchDebounceMs: 1500,
    intervalMinutes: number,
    sessions: { deltaBytes: 100_000, deltaMessages: 50 }
  },
  query: {
    maxResults: 6,
    minScore: 0.35,
    hybrid: {
      enabled: true,
      vectorWeight: 0.7,
      textWeight: 0.3,
      candidateMultiplier: 4,
      mmr: { enabled: false, lambda: 0.7 },
      temporalDecay: { enabled: false, halfLifeDays: 30 }
    }
  },
  cache: { enabled: true, maxEntries?: number },
  store: {
    driver: "sqlite",
    path: "{agentId}.sqlite",
    vector: { enabled: true, extensionPath?: string }
  }
}
```

---

## 11. Data Flow: End-to-End

```
                    WRITE PATH
                    =========

Agent conversation -> learns new fact
  |
  v
Agent writes to memory/YYYY-MM-DD.md  (or MEMORY.md)
  |
  v
Chokidar file watcher detects change
  |
  v
Debounce 1500ms -> marks dirty = true
  |
  v
Next sync trigger (search/interval/manual)
  |
  v
syncMemoryFiles():
  - List all memory files
  - Compare hashes with files table
  - For changed files:
    - chunkMarkdown(content, {tokens:400, overlap:80})
    - Check embedding_cache
    - embed(chunks) via provider
    - Store chunks + embeddings + FTS
  - Delete stale entries


                    READ PATH
                    =========

Agent needs to recall information
  |
  v
memory_search tool invoked with query
  |
  v
Async sync if dirty (flush latest changes)
  |
  v
Query expansion (stop words for 7 languages, CJK tokenization)
  |
  v
Parallel: vector search (sqlite-vec) + keyword search (FTS5)
  |
  v
Merge hybrid results (70% vector + 30% keyword)
  |
  v
Optional: temporal decay + MMR re-ranking
  |
  v
Return top-K results with citations


              COMPACTION PATH
              ===============

Session nears context limit
  |
  v
Memory flush triggered (pre-compaction)
  |
  v
Agent reviews about-to-be-compacted messages
  |
  v
Agent writes important info to memory/YYYY-MM-DD.md
  |
  v
Pi's compaction runs:
  - findCutPoint() identifies what to discard
  - generateSummary() creates LLM summary
  - compact() replaces old messages with summary
  |
  v
File watcher picks up new memory file changes
  |
  v
Next sync indexes the new memories
```

---

## 12. Ownership Map: Pi vs OpenClaw vs User

| Component | Owner | Location |
|-----------|-------|----------|
| SessionManager (JSONL, branching) | **Pi Agent** | `@mariozechner/pi-agent-core` |
| Compaction engine | **Pi Agent** | `@mariozechner/pi-agent-core` |
| Memory workspace files | **User** | `MEMORY.md`, `memory/*.md` |
| SQLite index + schema | **OpenClaw** | `src/memory/` |
| Embedding pipeline | **OpenClaw** | `src/memory/embeddings*.ts` |
| Hybrid search | **OpenClaw** | `src/memory/hybrid.ts`, `manager-search.ts` |
| Memory flush | **OpenClaw** | `src/auto-reply/reply/memory-flush.ts` |
| Context pruning | **OpenClaw** | `src/agents/pi-extensions/context-pruning/` |
| Session write lock | **OpenClaw** | `src/agents/session-write-lock.ts` |
| Memory search tool | **OpenClaw** | `src/agents/tools/memory-tool.ts` |
| Sync engine | **OpenClaw** | `src/memory/manager-sync-ops.ts` |
| QMD backend | **OpenClaw** | `src/memory/qmd-*.ts` |
| Memory search config | **User** (via config) | Agent YAML / OpenClaw config |

---

## 13. Design Rationale

### Why files as source of truth (not database)?
- **Human-readable**: Users can read/edit memory files directly
- **Version-controllable**: Memory can be tracked in git
- **Portable**: Copy files to transfer agent memory between machines
- **Resilient**: Index corruption doesn't lose data; just rebuild

### Why per-agent SQLite (not shared database)?
- **Isolation**: Each agent's knowledge is independent
- **Simplicity**: No connection pooling, no migrations across agents
- **Performance**: Small databases are fast; no index bloat from unrelated data

### Why hybrid search (not just embeddings)?
- **Cost**: FTS5 is free; embeddings cost API calls
- **Speed**: Keyword search is instant; vector search requires embedding the query
- **Accuracy**: Some queries are better served by exact match (project names, dates)
- **Fallback**: System works (degraded) even without embedding provider

### Why memory flush before compaction?
- **Information preservation**: Nothing is lost during compaction
- **Self-improving**: Agent learns to write better memories over time
- **Temporal trail**: Date-stamped files create a knowledge timeline
- **User control**: Users can see, edit, and delete specific memories

### Why atomic reindex with temp DB swap?
- **Crash safety**: Process crash during reindex doesn't corrupt the live index
- **Zero downtime**: The old index serves queries until the new one is ready
- **Rollback**: If reindex fails, the original is restored automatically

---

## 14. Key Files Reference

| Category | File | Purpose |
|----------|------|---------|
| **Core** | `src/memory/manager.ts` | Public API: search, readFile, status |
| **Core** | `src/memory/manager-sync-ops.ts` | Sync, reindex, file watchers |
| **Core** | `src/memory/manager-embedding-ops.ts` | Embedding generation, caching, retry |
| **Schema** | `src/memory/memory-schema.ts` | Database table definitions |
| **Search** | `src/memory/manager-search.ts` | Vector and keyword search |
| **Search** | `src/memory/hybrid.ts` | Result merging, BM25 scoring |
| **Search** | `src/memory/query-expansion.ts` | Stop words, tokenization |
| **Ranking** | `src/memory/temporal-decay.ts` | Exponential age decay |
| **Ranking** | `src/memory/mmr.ts` | Maximal marginal relevance |
| **Embeddings** | `src/memory/embeddings.ts` | Provider factory, normalization |
| **Embeddings** | `src/memory/embeddings-openai.ts` | OpenAI provider |
| **Embeddings** | `src/memory/embeddings-gemini.ts` | Gemini provider |
| **Embeddings** | `src/memory/embeddings-voyage.ts` | Voyage provider |
| **Batch** | `src/memory/batch-openai.ts` | OpenAI batch API |
| **Infra** | `src/memory/sqlite.ts` | Node SQLite wrapper |
| **Infra** | `src/memory/sqlite-vec.ts` | Vector extension loader |
| **Infra** | `src/memory/internal.ts` | Chunking, hashing, file entries |
| **Sessions** | `src/memory/session-files.ts` | Session transcript parsing |
| **Sync** | `src/memory/sync-index.ts` | Index entry management |
| **Sync** | `src/memory/sync-memory-files.ts` | Memory file sync |
| **Sync** | `src/memory/sync-session-files.ts` | Session file sync |
| **QMD** | `src/memory/qmd-manager.ts` | External search sidecar |
| **Tool** | `src/agents/tools/memory-tool.ts` | Agent-facing search tool |
| **Flush** | `src/auto-reply/reply/memory-flush.ts` | Pre-compaction persistence |
| **Pruning** | `src/agents/pi-extensions/context-pruning/` | Mid-conversation pruning |
| **Lock** | `src/agents/session-write-lock.ts` | Concurrent write prevention |
| **Security** | `src/memory/remote-http.ts` | Guarded remote HTTP (SSRF protection) |
| **Limits** | `src/memory/embedding-model-limits.ts` | Per-provider input token caps |
| **Config** | `src/agents/memory-search.ts` | Configuration resolution |
| **Config** | `src/config/byte-size.ts` | Shared byte-size parsing for flush thresholds |
