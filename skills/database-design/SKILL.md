---
name: database-design
description: "Use when designing a database schema for a new system, or when an existing schema needs to be revised. Two modes: (1) general schema design for any application, (2) AI-specific schema design for systems with conversation history, embeddings, agent state, or LLM outputs. Guides through data modeling decisions, type choices, indexing, and AI-specific storage patterns."
---

# Database Design

## The Core Problem

Bad schema design is expensive to fix later. The most common mistakes are: modeling the wrong entities, choosing the wrong database type for the use case, not thinking about query patterns before designing tables, and — for AI systems specifically — not accounting for the unique storage requirements of conversations, embeddings, and agent state.

**The discipline:** Understand the query patterns before designing the schema. The schema should make your most common queries fast and simple.

---

## Two Modes

**Mode 1: General Schema Design** — designing a schema for any application → Start at Step 1

**Mode 2: AI-Specific Schema Design** — system involves conversations, embeddings, agent state, or LLM outputs → Start at Step 1, then continue to Step A1

---

# Mode 1: General Schema Design

## Step 1: Understand the Data Before Modeling It

Before creating any tables or collections, answer these questions:

```
What are the core entities in this system?
(e.g. users, products, orders, sessions)

What are the relationships between them?
(one-to-one, one-to-many, many-to-many)

What are the most common read queries?
(e.g. "get all orders for a user", "get all products in a category")

What are the most common write operations?
(e.g. "create a new order", "update a product's price")

What data is read together most often?
(this determines what should be joined vs. denormalized)

What is the expected scale?
(hundreds of rows? millions? this affects indexing strategy)
```

Do not design any tables until you can answer all of these.

---

## Step 2: Choose the Right Database Type

| Database Type | Use When | Avoid When |
|---|---|---|
| **Relational (PostgreSQL, MySQL, SQLite)** | Data has clear structure and relationships. Transactions matter. Queries are complex (joins, aggregations). | Data is highly variable in structure. Schema changes frequently. |
| **Document (MongoDB, Firestore)** | Data structure varies per record. You read entire documents at once. Schema evolves frequently. | You need complex joins. Transactions across documents are common. |
| **Vector (Pinecone, pgvector, Chroma)** | You need semantic similarity search. You're storing embeddings. | You don't need similarity search — don't add vector DB just because you're using AI. |
| **Key-Value (Redis, DynamoDB)** | Simple lookups by ID. Caching. Session storage. High throughput. | Complex queries. Relationships between data. |
| **Time-series (TimescaleDB, InfluxDB)** | Data is append-only and time-ordered. Metrics, logs, events. | General-purpose data storage. |

**For most AI applications:** PostgreSQL + pgvector covers 80% of cases — relational data, vector search, and JSON storage in one database.

---

## Step 3: Design the Schema

### Naming conventions
- Tables: plural snake_case (`users`, `chat_sessions`, `agent_runs`)
- Columns: singular snake_case (`user_id`, `created_at`, `is_active`)
- Primary keys: `id` (UUID preferred over auto-increment for distributed systems)
- Foreign keys: `{table_singular}_id` (`user_id`, `session_id`)
- Timestamps: always include `created_at`, add `updated_at` for mutable records

### Required fields for every table
```sql
id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
updated_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()  -- only if records are mutable
```

### Design for your queries, not your entities

Wrong approach: model every entity as a table, then figure out queries later.
Right approach: write your 5 most common queries first, then design the schema that makes those queries simple.

```sql
-- Write this first:
-- "Get all messages in a conversation, ordered by time"
SELECT * FROM messages WHERE session_id = $1 ORDER BY created_at ASC;

-- Then design the table to support it:
CREATE TABLE messages (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES sessions(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    ...
    -- index on session_id + created_at for this query
);
CREATE INDEX idx_messages_session_created ON messages(session_id, created_at);
```

---

## Step 4: Indexing Strategy

**Index columns you filter or sort by frequently.** Every index speeds up reads but slows down writes.

```sql
-- Single column index
CREATE INDEX idx_users_email ON users(email);

-- Composite index (order matters — match your most common query's WHERE/ORDER BY pattern)
CREATE INDEX idx_messages_session_created ON messages(session_id, created_at);

-- Partial index (when you only query a subset)
CREATE INDEX idx_active_sessions ON sessions(user_id) WHERE is_active = true;
```

**Rules:**
- Always index foreign keys
- Always index columns used in WHERE clauses on large tables
- Never index columns with very low cardinality (e.g. boolean flags) unless combined with a selective column
- Don't over-index — each index adds write overhead

---

## Step 5: Schema Review Checklist

Before finalizing:

- [ ] Every table has `id`, `created_at`
- [ ] Foreign keys are indexed
- [ ] Most common queries are fast (no full table scans on large tables)
- [ ] No data is duplicated unnecessarily
- [ ] Nullable columns are intentional (every NULL should have a reason)
- [ ] Enum values are either a proper ENUM type or a foreign key to a lookup table
- [ ] Large text/JSON fields that won't be queried are stored as TEXT/JSONB, not VARCHAR(255)

---

---

# Mode 2: AI-Specific Schema Design

Use this in addition to Mode 1 when your system involves any of:
- Conversation history (chat messages, turns)
- LLM outputs (generated text, structured responses)
- Embeddings (vector representations)
- Agent state (memory, context, intermediate results)
- Evaluation and feedback (human ratings, LLM-as-judge scores)

---

## Step A1: Conversation and Message Storage

The standard pattern for storing conversations:

```sql
-- Sessions: one per conversation
CREATE TABLE sessions (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id      UUID REFERENCES users(id),
    created_at   TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at   TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    metadata     JSONB DEFAULT '{}'  -- flexible: store model used, config, etc.
);

-- Messages: one per turn
CREATE TABLE messages (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id   UUID NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
    role         TEXT NOT NULL CHECK (role IN ('user', 'assistant', 'system', 'tool')),
    content      TEXT NOT NULL,
    created_at   TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    token_count  INTEGER,             -- track usage
    metadata     JSONB DEFAULT '{}'   -- model, latency, finish_reason, etc.
);

CREATE INDEX idx_messages_session_created ON messages(session_id, created_at);
```

**Key decisions:**
- Store raw content, not processed versions — you may need to reprocess later
- Use `JSONB` for metadata — model versions, latency, tool calls, etc. change over time
- Include `token_count` — you will want this for cost tracking
- `ON DELETE CASCADE` on messages — if session is deleted, messages go with it. **Caution:** cascading deletes are powerful — accidental session deletion will remove all messages. Consider soft deletes (`is_deleted` flag) for production systems

---

## Step A2: LLM Output Storage

When you need to store and audit LLM outputs separately from conversation messages:

```sql
CREATE TABLE llm_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID REFERENCES sessions(id),
    node_name       TEXT,                    -- which agent/node produced this
    model           TEXT NOT NULL,           -- e.g. "gpt-4o", "claude-3-5-sonnet"
    input_tokens    INTEGER,
    output_tokens   INTEGER,
    latency_ms      INTEGER,
    input           JSONB NOT NULL,          -- full prompt/messages sent
    output          JSONB NOT NULL,          -- full response received
    parsed_output   JSONB,                   -- structured output if applicable
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    error           TEXT                     -- null if successful
);

CREATE INDEX idx_llm_runs_session ON llm_runs(session_id);
CREATE INDEX idx_llm_runs_node ON llm_runs(node_name, created_at);
```

**Why store full input/output:** You will need this for debugging, evaluation, and fine-tuning. Storage is cheap; missing data is not recoverable.

---

## Step A3: Embedding Storage

For semantic search and retrieval:

```sql
-- Using pgvector (PostgreSQL extension)
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE embeddings (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- what this embedding represents
    source_type  TEXT NOT NULL,   -- e.g. "document", "message", "artifact"
    source_id    UUID NOT NULL,
    -- the embedding
    model        TEXT NOT NULL,   -- e.g. "text-embedding-3-small"
    dimensions   INTEGER NOT NULL,
    embedding    vector(1536),    -- MUST match your embedding model's output dimensions
    -- Common dimensions: OpenAI text-embedding-3-small = 1536, Cohere embed-v3 = 1024,
    -- nomic-embed-text = 768. Check your model's docs before setting this.
    -- metadata for filtering
    metadata     JSONB DEFAULT '{}',
    created_at   TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- HNSW index for fast approximate nearest neighbor search
CREATE INDEX idx_embeddings_vector ON embeddings
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- Index for filtering by source type
CREATE INDEX idx_embeddings_source ON embeddings(source_type, source_id);
```

**Key decisions:**
- Store `model` and `dimensions` — when you switch embedding models, you'll need to regenerate. Knowing which model produced each embedding makes this manageable.
- Use HNSW index for production (faster queries). Use IVFFlat for very large datasets (>1M rows).
- Filter before vector search when possible — add WHERE clauses on metadata fields to reduce the search space.

**Similarity search query:**
```sql
SELECT source_id, 1 - (embedding <=> $1::vector) AS similarity
FROM embeddings
WHERE source_type = 'document'
ORDER BY embedding <=> $1::vector
LIMIT 10;
```

---

## Step A4: Agent State Storage

For agents that need to persist state across multiple runs or sessions:

```sql
CREATE TABLE agent_state (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id   UUID NOT NULL REFERENCES sessions(id),
    agent_name   TEXT NOT NULL,
    state        JSONB NOT NULL,      -- full state snapshot
    version      INTEGER NOT NULL DEFAULT 1,
    created_at   TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Get latest state for a session + agent
CREATE INDEX idx_agent_state_session_agent ON agent_state(session_id, agent_name, version DESC);
```

**Pattern:** append-only state snapshots. Never update in place — always insert a new row with `version + 1`. This gives you:
- Full audit trail of state evolution
- Ability to replay from any checkpoint
- Easy debugging (compare state at different points)

---

## Step A5: Evaluation and Feedback Storage

For tracking quality over time:

```sql
CREATE TABLE evaluations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id          UUID REFERENCES llm_runs(id),
    evaluator_type  TEXT NOT NULL CHECK (evaluator_type IN ('human', 'llm', 'rule')),
    evaluator_id    TEXT,            -- user ID or model name
    metric          TEXT NOT NULL,   -- e.g. "correctness", "relevance", "helpfulness"
    score           NUMERIC(4,3),    -- 0.000 to 1.000
    reasoning       TEXT,
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_evaluations_run ON evaluations(run_id);
CREATE INDEX idx_evaluations_metric ON evaluations(metric, created_at);
```

---

## AI Schema Checklist

Before finalizing an AI system schema:

- [ ] Conversation history stores `role`, `content`, `token_count`, and `metadata`
- [ ] LLM runs store full input and output (not just final answer)
- [ ] Embedding model name and dimensions are stored alongside the vector
- [ ] Agent state is append-only with version tracking
- [ ] Cost tracking is possible from stored token counts
- [ ] Evaluation scores can be linked back to specific LLM runs
- [ ] `JSONB` is used for fields that will evolve (metadata, config, parsed outputs)
- [ ] pgvector or equivalent is set up if embeddings are needed

---

## Common AI Schema Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Storing only the final answer, not the full LLM input/output | Can't debug failures, can't evaluate, can't fine-tune | Store full input and output in `llm_runs` |
| Not storing the embedding model name | Can't tell which embeddings need regeneration when you switch models | Always store `model` alongside the vector |
| Mutating agent state in place | Lose history, can't replay, hard to debug | Append-only state with version numbers |
| Using VARCHAR(255) for LLM outputs | Truncation on long outputs | Use TEXT — LLM outputs have no predictable max length |
| No token count tracking | Can't monitor costs, can't enforce limits | Add `token_count` to messages and `input_tokens`/`output_tokens` to llm_runs |
| One table for all message types | Can't query efficiently by role or type | Use `role` column with CHECK constraint |