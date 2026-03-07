# Design Doc 05: Memory System

## Overview

The memory system provides persistent, searchable storage for each agent. It is a three-layer pipeline: **write path** (store new memories), **read path** (recall relevant memories before each turn), and **compaction path** (flush in-context memories to storage at context limit). Storage is SQLite with hybrid BM25 + vector search. Each agent gets its own isolated database file.

## Core Concept

The LLM has a finite context window. Without memory, an agent forgets everything between sessions. With memory:
- Facts, preferences, decisions are stored as embeddings + BM25-indexed text
- Before each turn, top-K memories relevant to the current message are injected into the system prompt
- When the context window fills, a compaction step extracts important facts and flushes them to the DB before truncating the conversation

**Physical isolation**: `~/.openclaw/state/memory/{agentId}.sqlite` — one file per agent ID, no shared tables.

---

## Data Model

```typescript
interface MemoryRecord {
  id: string;                // UUID
  agentId: string;
  content: string;           // the stored fact/memory text
  embedding: Float32Array;   // vector from embedding model
  category: MemoryCategory;
  importance: number;        // 0.0–1.0, set by agent or scoring heuristic
  sourceType: MemorySource;
  sourceTurnId?: string;     // which conversation turn it came from
  createdAt: number;         // epoch ms
  updatedAt: number;
  accessedAt: number;        // last recall time (for LRU decay)
  accessCount: number;
  tags: string[];
  metadata: Record<string, unknown>;
}

type MemoryCategory =
  | "fact"         // factual knowledge: "User prefers dark mode"
  | "preference"   // behavioral preference: "Always use TypeScript"
  | "decision"     // past decision: "We chose PostgreSQL over MySQL on 2024-03-01"
  | "task"         // ongoing task state
  | "relationship" // info about people/orgs
  | "skill"        // learned capability
  | "episodic";    // event that happened

type MemorySource =
  | "explicit"     // agent called memory_store with intent
  | "compaction"   // extracted during context compaction
  | "inferred"     // auto-detected from conversation (future)
  | "imported";    // loaded from external source
```

---

## SQLite Schema

```sql
CREATE TABLE memories (
  id TEXT PRIMARY KEY,
  agent_id TEXT NOT NULL,
  content TEXT NOT NULL,
  embedding BLOB,              -- Float32Array serialized as binary
  category TEXT NOT NULL,
  importance REAL DEFAULT 0.5,
  source_type TEXT NOT NULL,
  source_turn_id TEXT,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  accessed_at INTEGER NOT NULL,
  access_count INTEGER DEFAULT 0,
  tags TEXT DEFAULT '[]',      -- JSON array
  metadata TEXT DEFAULT '{}'   -- JSON object
);

-- BM25 full-text search
CREATE VIRTUAL TABLE memories_fts USING fts5(
  id UNINDEXED,
  content,
  tokenize = 'porter unicode61'
);

-- Indexes for common queries
CREATE INDEX idx_memories_agent_id ON memories(agent_id);
CREATE INDEX idx_memories_category ON memories(agent_id, category);
CREATE INDEX idx_memories_importance ON memories(agent_id, importance DESC);
CREATE INDEX idx_memories_accessed ON memories(agent_id, accessed_at DESC);
```

---

## Write Path

```typescript
class MemoryStore {
  private db: Database;
  private embedder: EmbeddingModel;

  constructor(private agentId: string, dbPath: string) {
    this.db = openDatabase(dbPath);
    this.embedder = getEmbeddingModel();
    this.runMigrations();
  }

  async store(params: {
    content: string;
    category: MemoryCategory;
    importance?: number;
    source?: MemorySource;
    tags?: string[];
    metadata?: Record<string, unknown>;
  }): Promise<MemoryRecord> {
    const embedding = await this.embedder.embed(params.content);
    const now = Date.now();

    const record: MemoryRecord = {
      id: crypto.randomUUID(),
      agentId: this.agentId,
      content: params.content,
      embedding,
      category: params.category,
      importance: params.importance ?? 0.5,
      sourceType: params.source ?? "explicit",
      createdAt: now,
      updatedAt: now,
      accessedAt: now,
      accessCount: 0,
      tags: params.tags ?? [],
      metadata: params.metadata ?? {},
    };

    this.db.prepare(`
      INSERT INTO memories
        (id, agent_id, content, embedding, category, importance, source_type,
         created_at, updated_at, accessed_at, access_count, tags, metadata)
      VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    `).run(
      record.id, record.agentId, record.content,
      serializeEmbedding(embedding),
      record.category, record.importance, record.sourceType,
      record.createdAt, record.updatedAt, record.accessedAt,
      record.accessCount,
      JSON.stringify(record.tags), JSON.stringify(record.metadata),
    );

    // Also insert into FTS index
    this.db.prepare(`
      INSERT INTO memories_fts(id, content) VALUES (?, ?)
    `).run(record.id, record.content);

    return record;
  }

  async forget(id: string): Promise<boolean> {
    const result = this.db.prepare(
      `DELETE FROM memories WHERE id = ? AND agent_id = ?`
    ).run(id, this.agentId);

    if (result.changes > 0) {
      this.db.prepare(`DELETE FROM memories_fts WHERE id = ?`).run(id);
      return true;
    }
    return false;
  }

  async update(id: string, updates: Partial<Pick<MemoryRecord, "content" | "importance" | "tags" | "metadata">>): Promise<boolean> {
    const existing = this.getById(id);
    if (!existing) return false;

    if (updates.content) {
      // Re-embed if content changed
      const embedding = await this.embedder.embed(updates.content);
      this.db.prepare(`UPDATE memories SET embedding = ? WHERE id = ?`)
        .run(serializeEmbedding(embedding), id);
      this.db.prepare(`UPDATE memories_fts SET content = ? WHERE id = ?`)
        .run(updates.content, id);
    }

    this.db.prepare(`
      UPDATE memories SET
        content = COALESCE(?, content),
        importance = COALESCE(?, importance),
        tags = COALESCE(?, tags),
        metadata = COALESCE(?, metadata),
        updated_at = ?
      WHERE id = ? AND agent_id = ?
    `).run(
      updates.content ?? null,
      updates.importance ?? null,
      updates.tags ? JSON.stringify(updates.tags) : null,
      updates.metadata ? JSON.stringify(updates.metadata) : null,
      Date.now(), id, this.agentId,
    );

    return true;
  }
}
```

---

## Read Path: Hybrid Search

```typescript
interface SearchResult {
  record: MemoryRecord;
  score: number;              // combined BM25 + cosine score
  bm25Score: number;
  vectorScore: number;
}

async function searchMemories(params: {
  query: string;
  agentId: string;
  limit?: number;
  category?: MemoryCategory;
  minImportance?: number;
  hybridWeight?: number;      // 0.0 = pure BM25, 1.0 = pure vector, default 0.5
}): Promise<SearchResult[]> {
  const {
    query, agentId, limit = 10, hybridWeight = 0.5,
    category, minImportance = 0.0,
  } = params;

  const store = getStore(agentId);

  // BM25 search
  const bm25Results = store.db.prepare(`
    SELECT m.id, m.content, m.category, m.importance,
           bm25(memories_fts) AS bm25_score
    FROM memories_fts
    JOIN memories m ON m.id = memories_fts.id
    WHERE memories_fts MATCH ?
      AND m.agent_id = ?
      ${category ? "AND m.category = ?" : ""}
      AND m.importance >= ?
    ORDER BY bm25_score
    LIMIT ?
  `).all(
    query, agentId,
    ...(category ? [category] : []),
    minImportance, limit * 2,  // fetch 2x for re-ranking
  ) as Array<{ id: string; bm25_score: number }>;

  // Vector search
  const queryEmbedding = await store.embedder.embed(query);
  const vectorResults = await vectorSearch({
    embedding: queryEmbedding,
    agentId,
    category,
    minImportance,
    limit: limit * 2,
    db: store.db,
  });

  // Merge and re-rank
  const combined = mergeHybridResults(bm25Results, vectorResults, hybridWeight);

  // Update access metadata for recalled memories
  const topK = combined.slice(0, limit);
  await updateAccessMetadata(store.db, topK.map((r) => r.record.id));

  return topK;
}

function mergeHybridResults(
  bm25: Array<{ id: string; bm25_score: number }>,
  vector: Array<{ id: string; vector_score: number; record: MemoryRecord }>,
  hybridWeight: number,
): SearchResult[] {
  // Normalize BM25 scores (BM25 returns negative values in SQLite FTS5)
  const maxBm25 = Math.max(...bm25.map((r) => Math.abs(r.bm25_score)), 1);
  const bm25Map = new Map(
    bm25.map((r) => [r.id, Math.abs(r.bm25_score) / maxBm25]),
  );

  // Merge by ID
  const allIds = new Set([...bm25.map((r) => r.id), ...vector.map((r) => r.id)]);
  const results: SearchResult[] = [];

  for (const id of allIds) {
    const bm25Score = bm25Map.get(id) ?? 0;
    const vectorEntry = vector.find((r) => r.id === id);
    const vectorScore = vectorEntry?.vector_score ?? 0;
    const record = vectorEntry?.record ?? getRecordById(id);
    if (!record) continue;

    const score =
      (1 - hybridWeight) * bm25Score + hybridWeight * vectorScore;

    results.push({ record, score, bm25Score, vectorScore });
  }

  return results.sort((a, b) => b.score - a.score);
}

// Naive vector search (cosine similarity over all records for agent)
// For production: use sqlite-vss or hnswlib
async function vectorSearch(params: {
  embedding: Float32Array;
  agentId: string;
  category?: MemoryCategory;
  minImportance: number;
  limit: number;
  db: Database;
}): Promise<Array<{ id: string; vector_score: number; record: MemoryRecord }>> {
  const rows = params.db.prepare(`
    SELECT id, content, embedding, category, importance, created_at, accessed_at, access_count, tags, metadata
    FROM memories
    WHERE agent_id = ?
      ${params.category ? "AND category = ?" : ""}
      AND importance >= ?
      AND embedding IS NOT NULL
  `).all(
    params.agentId,
    ...(params.category ? [params.category] : []),
    params.minImportance,
  ) as Array<{ id: string; embedding: Buffer; [k: string]: unknown }>;

  return rows
    .map((row) => {
      const embedding = deserializeEmbedding(row.embedding);
      const score = cosineSimilarity(params.embedding, embedding);
      return { id: row.id, vector_score: score, record: rowToRecord(row) };
    })
    .sort((a, b) => b.vector_score - a.vector_score)
    .slice(0, params.limit);
}

function cosineSimilarity(a: Float32Array, b: Float32Array): number {
  let dot = 0, normA = 0, normB = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }
  return dot / (Math.sqrt(normA) * Math.sqrt(normB) + 1e-8);
}
```

---

## Compaction Path: Flush to DB

When context window is near capacity, the compaction process extracts memories before truncating:

```typescript
async function flushContextToMemory(params: {
  agentId: string;
  conversationHistory: Message[];
  cfg: AgentConfig;
}): Promise<{ flushedCount: number; summary: string }> {
  const { agentId, conversationHistory } = params;

  // Ask the LLM to extract key facts from the conversation
  const extractionPrompt = buildExtractionPrompt(conversationHistory);
  const extracted = await callLLM({
    model: params.cfg.compactionModel ?? params.cfg.model,
    system: EXTRACTION_SYSTEM_PROMPT,
    messages: [{ role: "user", content: extractionPrompt }],
    maxTokens: 2000,
  });

  // Parse structured memory entries from extraction output
  const memories = parseExtractedMemories(extracted);
  const store = getStore(agentId);

  let flushedCount = 0;
  for (const mem of memories) {
    await store.store({
      content: mem.content,
      category: mem.category,
      importance: mem.importance,
      source: "compaction",
      tags: mem.tags,
    });
    flushedCount++;
  }

  // Build a summary to prepend to the truncated context
  const summary = buildCompactionSummary(memories);
  return { flushedCount, summary };
}

const EXTRACTION_SYSTEM_PROMPT = `
Extract key facts, decisions, preferences, and relationships from the conversation.
Output JSON array with schema:
[{ "content": string, "category": "fact"|"preference"|"decision"|"relationship", "importance": 0.0-1.0, "tags": string[] }]
Only extract information worth remembering long-term. Skip small talk and transient details.
`.trim();

function buildCompactionSummary(memories: ExtractedMemory[]): string {
  if (memories.length === 0) return "";
  const lines = ["[Context compacted. Key facts preserved in memory:]"];
  for (const m of memories.slice(0, 5)) {
    lines.push(`- ${m.content}`);
  }
  if (memories.length > 5) lines.push(`- ...and ${memories.length - 5} more`);
  return lines.join("\n");
}
```

---

## Memory Injection into System Prompt

Before each turn, search memories relevant to the current user message and inject top-K into the `workspace_memory` section:

```typescript
async function recallRelevantMemories(params: {
  userMessage: string;
  agentId: string;
  maxExcerpts?: number;
  maxTokens?: number;
}): Promise<string[]> {
  const { userMessage, agentId, maxExcerpts = 5, maxTokens = 800 } = params;

  const results = await searchMemories({
    query: userMessage,
    agentId,
    limit: maxExcerpts * 2,  // over-fetch, then trim by token budget
  });

  const excerpts: string[] = [];
  let tokenCount = 0;

  for (const result of results) {
    if (result.score < 0.3) break; // below relevance threshold
    const excerpt = formatMemoryExcerpt(result);
    const tokens = estimateTokens(excerpt);
    if (tokenCount + tokens > maxTokens) break;
    excerpts.push(excerpt);
    tokenCount += tokens;
  }

  return excerpts;
}

function formatMemoryExcerpt(result: SearchResult): string {
  const { record, score } = result;
  const age = formatRelativeTime(record.createdAt);
  return `[${record.category}, ${age}, relevance: ${(score * 100).toFixed(0)}%] ${record.content}`;
}
```

---

## Tools Exposed to Agent

```typescript
const memoryTools: AgentTool[] = [
  {
    name: "memory_search",
    description: "Search your persistent memory for relevant past context",
    groups: ["memory"],
    inputSchema: {
      type: "object",
      properties: {
        query: { type: "string", description: "Search query" },
        category: { type: "string", enum: ["fact", "preference", "decision", "relationship", "task"] },
        limit: { type: "number", default: 5 },
      },
      required: ["query"],
    },
    handler: async (input) => {
      const { query, category, limit } = input as { query: string; category?: MemoryCategory; limit?: number };
      return searchMemories({ query, agentId: currentAgentId(), category, limit });
    },
  },
  {
    name: "memory_store",
    description: "Store a fact, preference, or decision in persistent memory",
    groups: ["memory"],
    inputSchema: {
      type: "object",
      properties: {
        content: { type: "string" },
        category: { type: "string", enum: ["fact", "preference", "decision", "relationship", "task", "episodic"] },
        importance: { type: "number", description: "0.0 to 1.0" },
        tags: { type: "array", items: { type: "string" } },
      },
      required: ["content", "category"],
    },
    handler: async (input) => {
      const store = getStore(currentAgentId());
      return store.store(input as Parameters<typeof store.store>[0]);
    },
  },
  {
    name: "memory_forget",
    description: "Remove a memory by ID",
    groups: ["memory"],
    inputSchema: {
      type: "object",
      properties: { id: { type: "string" } },
      required: ["id"],
    },
    handler: async (input) => {
      const store = getStore(currentAgentId());
      return { deleted: await store.forget((input as { id: string }).id) };
    },
  },
];
```

---

## Implementation Checklist

- [ ] SQLite schema: `memories` table + `memories_fts` FTS5 virtual table
- [ ] Per-agent DB path: `~/.openclaw/state/memory/{agentId}.sqlite`
- [ ] `MemoryStore.store()` — embed + insert into both tables
- [ ] `MemoryStore.forget()` — delete from both tables
- [ ] `MemoryStore.update()` — re-embed if content changed
- [ ] `vectorSearch()` — cosine similarity over all agent records (naive for small sets)
- [ ] BM25 search via FTS5 MATCH query
- [ ] `mergeHybridResults()` — normalize BM25 + vector scores, weight-sum
- [ ] `searchMemories()` — combined API with `hybridWeight` param
- [ ] Access metadata update on recall (`accessed_at`, `access_count`)
- [ ] `flushContextToMemory()` — LLM extraction + bulk store on compaction
- [ ] `recallRelevantMemories()` — top-K for system prompt injection, token-budgeted
- [ ] `memory_search`, `memory_store`, `memory_forget` tools
- [ ] LRU decay scoring (deprioritize stale, never-recalled memories)
- [ ] `MemoryCategory` and `MemorySource` enums
- [ ] Embedding model abstraction (default: local model, configurable)
