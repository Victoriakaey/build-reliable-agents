---
name: agent-architecture
description: "Use at the start of any LLM agent project, or when reconsidering an existing architecture. Guides decisions across four layers: workflow vs agent, single-agent vs multi-agent, tool-use vs specialized nodes, and retrieval strategy. Each decision has concrete tradeoffs and a recommended default."
---

# Agent Architecture Decision Framework

## Overview

Agent architecture decisions compound. A wrong early decision (e.g. single agent with tool-use when you need observability) becomes expensive to fix later. Make these four decisions explicitly, in order, before writing any code.

---

## Decision 1: Workflow vs Agent

**The question:** Does your pipeline need to recover from mid-step failures, or is it always the same sequence?

| | Workflow | Agent |
|---|---|---|
| Structure | Fixed sequence, always A→B→C→D | Can loop back: if B fails, retry A with different input |
| Use when | Steps never need to reorder or retry | Failures in one step should inform retries of earlier steps |
| Observability | High — path is always the same | Medium — path varies by run |
| Debugging | Easy — failure is always at a known step | Harder — need to trace which path was taken |

**Default recommendation:** Agent, unless your pipeline is truly linear with no recovery needed. The ability to route back on failure (e.g. SQL validation failure → regenerate SQL) is almost always worth the small added complexity.

---

## Decision 2: Single Agent vs Multi-Agent

**The question:** Do your components need independent state and memory, or can they share state?

| | Single Agent (shared state) | Multi-Agent |
|---|---|---|
| State | One shared state across all nodes | Each agent has own state and memory |
| Communication | Direct via shared state dict | Message passing or shared memory |
| Use when | Components are steps in one pipeline | Components are truly independent actors with different goals |
| Complexity | Lower | Higher |
| Latency | Lower | Higher (inter-agent communication) |
| Debugging | Easier — one state to inspect | Harder — must trace across agent boundaries |

**Default recommendation:** Single agent with specialized nodes. For a single-pipeline system, multi-agent adds complexity that rarely pays off. Use multi-agent when you genuinely need independent memory per component, or when components run concurrently with different LLM models and goals.

**The overkill test:** If your "multi-agent" system has one orchestrator directing all other agents sequentially, it's a single agent with extra steps. Flatten it.

---

## Decision 3: Tool-Use vs Specialized Nodes

**The question:** Should one LLM decide which tools to use, or should each step be a dedicated node?

| | Single Agent + Tool-Use | Specialized Nodes |
|---|---|---|
| Control | LLM decides which tool to call and when | You control the sequence explicitly |
| Observability | Low — hard to know which tool ran and why | High — each node is a named, traceable step |
| Step ordering | Not guaranteed — LLM may skip or reorder | Guaranteed — graph topology enforces order |
| Error attribution | Hard — which tool call failed? | Easy — which node failed? |
| Flexibility | High — LLM adapts to unexpected inputs | Medium — unexpected cases need explicit routing |
| Use when | Task is open-ended, steps unknown in advance | Task has known sequence of steps |

**Default recommendation:** Specialized nodes for any pipeline with known steps. Tool-use is better for open-ended tasks where you genuinely don't know what steps are needed (e.g. a research agent that might need 2 or 20 searches).

**The sequence test:** If you can write down the steps your pipeline takes before running it, use specialized nodes. If the steps only become clear at runtime, use tool-use.

---

## Decision 4: Retrieval Strategy

**The question:** How does your agent get information from its data source?

| Strategy | Use When | Avoid When |
|---|---|---|
| Text-to-SQL | Data is structured (tables, rows, columns). Questions can be answered by querying known fields. | Data is unstructured prose. Schema is unknown or changes frequently. |
| RAG (vector search) | Data is unstructured text. Questions require semantic similarity ("find things like X"). | Data is structured. Exact answers are needed. SQL can express the query. |
| Hybrid (SQL + FTS) | Structured data with some free-text fields. Structured queries for enumerable fields, FTS for text search. | When either alone would work — don't add complexity without need. |
| Hardcoded templates | Query patterns are known and fixed. Determinism is critical. | Query variety is high. Schema changes frequently. |

**For SQL-based retrieval, MUST implement these safety layers:**
1. **Syntax validation** before execution (catch LLM SQL errors early)
2. **Destructive keyword blocking** (regex on stripped quoted content — block DROP, DELETE, UPDATE, INSERT, ALTER, TRUNCATE)
3. **Read-only connection** (defense in depth even after validation)
4. **Row cap** at Python level, not just in prompt (LLM may omit LIMIT clause)

---

## Decision 5: Where to Put Intelligence

For each step in your pipeline, decide: **LLM or deterministic?**

| Step Type | Prefer LLM | Prefer Deterministic |
|---|---|---|
| Understanding intent | ✓ | |
| Decomposing a question | ✓ | |
| Generating SQL or queries | ✓ | |
| Validating SQL syntax | | ✓ (use EXPLAIN QUERY PLAN) |
| Blocking destructive keywords | | ✓ (regex) |
| Routing based on a score | | ✓ (derive from numeric field) |
| Formatting output | | ✓ |
| Enforcing row caps | | ✓ (cursor.fetchmany()) |
| Judging quality | ✓ | |

**Principle:** Every deterministic component you add reduces variance and improves debuggability. Never use an LLM for something a simple function can do.

---

## Step-by-Step Architecture Decision Process

When starting a new project:

1. **Write down the pipeline steps in order** (even if approximate)
2. **Apply Decision 1** (workflow vs agent) — does any step need to retry an earlier step?
3. **Apply Decision 2** (single vs multi) — do any components need independent state?
4. **Apply Decision 3** (tool-use vs nodes) — is the step sequence known in advance?
5. **Apply Decision 4** (retrieval strategy) — what is the data source and query type?
6. **Apply Decision 5** (LLM vs deterministic) — for each step, which is appropriate?
7. **Draw the graph** — nodes, edges, routing conditions, loop guards
8. **Identify the blast radius** of each LLM node (what downstream nodes consume its output?)
9. **Write the loop guards** — every cycle in the graph needs a hard exit condition

---

## Architecture Evolution Warning

Systems evolve. The architecture that was right at design time may not be right after:
- New query types require new routing
- A simple node grows into a multi-step process
- An LLM node turns out to need deterministic behavior

**Watch for these signals that architecture needs revisiting:**
- A node's prompt is over 100 lines
- A single node is doing more than one conceptually distinct thing
- Debugging a failure requires tracing through more than 3 nodes to find root cause
- Adding a new feature requires changing 4+ files

When these appear, revisit the relevant decision rather than adding more complexity on top.