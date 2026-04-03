---
name: memory-system
description: "Use when designing how an agent remembers and uses information across turns, sessions, or runs. Covers the four types of agent memory (in-context, episodic, semantic, procedural), when to use each, how to manage context window limits, retrieval strategies, and memory decay. Cross-references database-design skill for storage schema patterns."
---

# Memory System Design

## The Core Problem

Most agents have no memory beyond the current context window. This means every conversation starts from scratch, the agent can't learn from past interactions, and users have to repeat themselves constantly. But adding memory naively — just appending everything to the context — hits token limits fast and degrades response quality with irrelevant history.

**The discipline:** Choose the right memory type for each kind of information. Not everything needs to be remembered, and not everything remembered needs to be in the context window.

---

## The Four Types of Agent Memory

Before designing anything, understand what kind of memory your agent actually needs.

| Memory Type | What It Stores | Scope | Retrieval |
|---|---|---|---|
| **In-context (working)** | Current conversation, immediate task state | Current session only | Always available — it's in the prompt |
| **Episodic** | Past conversations, past events, interaction history | Across sessions | Retrieved by recency or similarity |
| **Semantic** | Facts, knowledge, user preferences, domain info | Persistent | Retrieved by semantic similarity |
| **Procedural** | How to do things — learned behaviors, improved prompts | Persistent | Applied at system level, not retrieved per query |

Most agents only need **in-context + one other type**. Adding all four is usually overkill.

---

## Step 1: Decide What Needs to Be Remembered

For each type of information in your system, ask:

**Does it need to persist beyond this conversation?**
- No → in-context memory is enough
- Yes → needs episodic or semantic storage

**Is it a fact/preference (static) or an event (happened at a point in time)?**
- Fact/preference → semantic memory
- Event/interaction → episodic memory

**Does it change how the agent behaves system-wide?**
- Yes → procedural memory (prompt updates, behavior rules)
- No → episodic or semantic

### Common patterns

| Situation | Memory Type Needed |
|---|---|
| Multi-turn chatbot within one session | In-context only |
| Chatbot that remembers past conversations | In-context + Episodic |
| Personal assistant that knows your preferences | In-context + Semantic |
| Agent that improves over time from feedback | In-context + Procedural |
| RAG system over a knowledge base | In-context + Semantic |
| Customer support bot with case history | In-context + Episodic + Semantic |

---

## Step 2: In-Context Memory (Working Memory)

This is the conversation history in the prompt. It's always available but limited by the context window.

### The token budget problem

Every token in the context window costs money and affects latency. Long conversation histories get expensive fast.

```
Example (as of early 2026 — check current pricing):
gpt-4o: ~$2.50 per 1M input tokens
A 50-turn conversation at 200 tokens/turn = 10,000 tokens = $0.025 per query
At 1,000 queries/day = $25/day just for conversation history
```

### Context window management strategies

**Strategy 1: Fixed window (simplest)**
Keep only the last N turns. Simple to implement, loses older context.

```python
def get_context_messages(messages: list, max_turns: int = 10) -> list:
    # always keep system message
    system = [m for m in messages if m["role"] == "system"]
    # keep last N turns (user + assistant pairs)
    recent = messages[-max_turns * 2:]
    return system + recent
```

**Strategy 2: Token budget**
Keep messages until you hit a token limit. More precise than turn count.

```python
import tiktoken

def get_context_within_budget(messages: list, max_tokens: int = 4000) -> list:
    enc = tiktoken.encoding_for_model("gpt-4o")
    system = [m for m in messages if m["role"] == "system"]
    non_system = [m for m in messages if m["role"] != "system"]

    selected = []
    token_count = sum(len(enc.encode(m["content"])) for m in system)

    for message in reversed(non_system):  # most recent first
        tokens = len(enc.encode(message["content"]))
        if token_count + tokens > max_tokens:
            break
        selected.insert(0, message)
        token_count += tokens

    return system + selected
```

**Strategy 3: Summarization**
When the context gets too long, summarize older turns into a compressed summary and keep recent turns in full.

```python
async def compress_context(messages: list, keep_recent: int = 6) -> list:
    if len(messages) <= keep_recent + 1:  # +1 for system
        return messages

    system = messages[0]  # system message
    to_summarize = messages[1:-keep_recent]
    recent = messages[-keep_recent:]

    summary = await llm.generate(
        f"Summarize this conversation history concisely:\n\n"
        + "\n".join(f"{m['role']}: {m['content']}" for m in to_summarize)
    )

    summary_message = {
        "role": "system",
        "content": f"[Earlier conversation summary: {summary}]"
    }

    return [system, summary_message] + recent
```

**Which strategy to use:**
- Simple chatbot → fixed window (10 turns is usually enough)
- Cost-sensitive production → token budget
- Long-running tasks where history matters → summarization

---

## Step 3: Episodic Memory (Past Interactions)

Stores what happened in past conversations. Retrieved by recency or relevance.

### Storage schema

See `database-design` skill — Step A1 (Conversation and Message Storage) for the full schema. Key fields:

```sql
sessions (id, user_id, created_at, metadata)
messages (id, session_id, role, content, created_at, token_count)
```

### Retrieval patterns

**Recency-based:** Get the last N sessions for a user.
```sql
SELECT s.id, s.created_at, m.content
FROM sessions s
JOIN messages m ON m.session_id = s.id
WHERE s.user_id = $1
ORDER BY s.created_at DESC
LIMIT 5;
```

**Similarity-based:** Find past conversations relevant to the current query.
```sql
-- Requires embeddings on messages (see database-design Step A3)
SELECT m.content, 1 - (e.embedding <=> $1::vector) AS similarity
FROM messages m
JOIN embeddings e ON e.source_id = m.id AND e.source_type = 'message'
WHERE m.role = 'user'
ORDER BY e.embedding <=> $1::vector
LIMIT 5;
```

### When to inject episodic memory

Inject retrieved past context as a system message before the current conversation:

```python
async def build_prompt_with_episodic_memory(
    user_id: str,
    current_query: str,
    system_prompt: str
) -> list:
    # retrieve relevant past interactions
    past = await retrieve_similar_sessions(user_id, current_query, limit=3)

    memory_context = ""
    if past:
        memory_context = "\n\nRelevant past interactions:\n"
        for session in past:
            memory_context += f"- {session['summary']}\n"

    return [
        {"role": "system", "content": system_prompt + memory_context},
        # current conversation follows
    ]
```

---

## Step 4: Semantic Memory (Facts and Knowledge)

Stores persistent facts — user preferences, domain knowledge, entity information. Retrieved by semantic similarity.

### What belongs in semantic memory

- User preferences ("prefers concise answers", "timezone is PST")
- Entity facts ("Acme Corp is a customer since 2023, uses the Pro plan")
- Domain knowledge ("our return policy is 30 days")
- Extracted entities from past conversations ("user mentioned they have 3 kids")

### Storage schema

```sql
CREATE TABLE memory_items (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id      UUID REFERENCES users(id),
    -- or agent_id, session_id, etc. depending on scope
    memory_type  TEXT NOT NULL,   -- 'preference', 'fact', 'entity', 'knowledge'
    content      TEXT NOT NULL,   -- the fact in plain text
    source       TEXT,            -- where this came from ('user_stated', 'extracted', 'inferred')
    -- Note: extraction prompts return memory_type values ('preference', 'fact', 'entity')
    -- source tracks HOW the memory was obtained, memory_type tracks WHAT it is
    confidence   NUMERIC(3,2),    -- 0.00 to 1.00
    last_accessed TIMESTAMP WITH TIME ZONE,
    access_count  INTEGER DEFAULT 0,
    created_at   TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at   TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Embeddings stored separately (see database-design Step A3)
```

### Memory extraction

Extract memories from conversations automatically:

```python
EXTRACTION_PROMPT = """
Review this conversation and extract any facts worth remembering about the user.
Focus on: preferences, personal details they shared, constraints they mentioned.
Return as JSON array: [{"content": "...", "type": "preference|fact|entity", "confidence": 0.0-1.0}]
Only extract clear, explicit information. Do not infer or guess.
If nothing worth remembering, return [].

Conversation:
{conversation}
"""

async def extract_memories(messages: list) -> list[dict]:
    conversation = "\n".join(f"{m['role']}: {m['content']}" for m in messages)
    try:
        response = await llm.generate(EXTRACTION_PROMPT.format(conversation=conversation))
        return json.loads(response)
    except (json.JSONDecodeError, Exception) as e:
        logging.warning(f"Memory extraction failed: {e}")
        return []  # fail open — missing memories are better than crashing
```

### Memory retrieval and injection

```python
async def get_relevant_memories(user_id: str, query: str, limit: int = 5) -> list:
    query_embedding = await embed(query)
    return await db.query("""
        SELECT m.content, m.memory_type
        FROM memory_items m
        JOIN embeddings e ON e.source_id = m.id
        WHERE m.user_id = $1
        ORDER BY e.embedding <=> $2::vector
        LIMIT $3
    """, user_id, query_embedding, limit)

def format_memories_for_prompt(memories: list) -> str:
    if not memories:
        return ""
    lines = [f"- {m['content']}" for m in memories]
    return "What you know about this user:\n" + "\n".join(lines)
```

---

## Step 5: Procedural Memory (Learned Behaviors)

Stores how to do things better — improved prompts, learned rules, behavior adjustments based on feedback.

### What belongs in procedural memory

- Prompt improvements learned from failures ("when user asks about X, always clarify Y first")
- Behavioral rules derived from feedback ("user prefers bullet points over paragraphs")
- Workflow improvements ("for this task type, always do step A before step B")

### Storage pattern

Procedural memory is typically applied at the **system prompt level**, not retrieved per query:

```sql
CREATE TABLE procedural_rules (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scope       TEXT NOT NULL,  -- 'global', 'user', 'task_type'
    scope_id    UUID,           -- user_id or null for global
    rule        TEXT NOT NULL,  -- the behavioral rule in plain text
    source      TEXT,           -- 'manual', 'feedback', 'learned'
    priority    INTEGER DEFAULT 0,
    is_active   BOOLEAN DEFAULT true,
    created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

```python
async def build_system_prompt_with_rules(user_id: str, base_prompt: str) -> str:
    rules = await db.query("""
        SELECT rule FROM procedural_rules
        WHERE is_active = true
        AND (scope = 'global' OR (scope = 'user' AND scope_id = $1))
        ORDER BY priority DESC
    """, user_id)

    if not rules:
        return base_prompt

    rules_text = "\n".join(f"- {r['rule']}" for r in rules)
    return base_prompt + f"\n\nBehavioral rules:\n{rules_text}"
```

---

## Step 6: Memory Maintenance and Cleanup

Memory without cleanup grows indefinitely and degrades retrieval quality.

### Decay strategies

**Time-based expiry:** Delete memories older than N days.
```sql
DELETE FROM memory_items
WHERE last_accessed < NOW() - INTERVAL '90 days'
AND memory_type != 'preference';  -- keep preferences longer
```

**Access-based:** Deprioritize memories that are never retrieved.
```sql
-- Mark low-value memories for review
UPDATE memory_items
SET is_active = false
WHERE access_count = 0
AND created_at < NOW() - INTERVAL '30 days';
```

**Confidence decay:** Reduce confidence of old facts that haven't been reinforced.
```sql
UPDATE memory_items
SET confidence = confidence * 0.9
WHERE updated_at < NOW() - INTERVAL '7 days'
AND memory_type = 'inferred';
```

### What to keep vs. expire

| Memory Type | Keep | Expire |
|---|---|---|
| User preferences | Indefinitely | Only if user contradicts them |
| Past conversations | 90 days (or summarize) | Raw messages after 30 days |
| Extracted facts | Until contradicted | Low-confidence inferences after 30 days |
| Procedural rules | Until manually removed | N/A |

---

## Memory System Checklist

- [ ] Identified which of the four memory types are needed
- [ ] Context window management strategy chosen and implemented
- [ ] Storage schema in place (see `database-design` skill for patterns)
- [ ] Retrieval strategy defined (recency vs. similarity vs. both)
- [ ] Memory injection point in prompt defined
- [ ] Memory extraction logic implemented (if extracting from conversations)
- [ ] Decay/cleanup strategy defined
- [ ] Token cost of memory injection estimated

---

## Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Injecting all memories every time | Context bloat, irrelevant info, high cost | Retrieve only relevant memories per query |
| Never expiring memories | Quality degrades over time, storage grows | Implement decay strategy |
| Storing memories without confidence scores | Can't distinguish strong facts from weak inferences | Always record confidence and source |
| Using one memory type for everything | Wrong retrieval strategy for different info types | Map each info type to the right memory category |
| Extracting memories in the hot path | Adds latency to every query | Extract memories asynchronously after the conversation |
| No memory for procedural learning | Agent never improves from feedback | Implement simple rule storage even if not ML-based |
| No deduplication on memory extraction | Same fact stored multiple times, wastes tokens on retrieval | Deduplicate by content similarity before inserting (cosine similarity > 0.95 = duplicate) |