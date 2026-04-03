---
name: harness-design
description: "Use when designing or improving the action space, observation format, tool boundaries, and evaluation signals of an LLM agent. Covers the five layers of harness design: what tools the agent has, what it sees after acting, how each tool is scoped, how behavior is evaluated, and how to iterate when the agent misbehaves. Prevents the most common harness failures — agents that pick wrong tools, ignore critical information, or loop without progress."
---

# Agent Harness Design

## The Core Problem

Most agent failures are not prompt failures — they are harness failures. The agent picks the wrong tool because the action space is ambiguous. It ignores critical data because the observation is buried in noise. It loops without progress because no signal tells it when to stop. These are not problems you can fix by rewriting the prompt. They require redesigning the harness: what the agent can do, what it sees, and how its behavior is evaluated.

**The discipline:** Design the harness before writing the prompt. The prompt adapts to the harness, not the other way around.

---

## The Five Layers

Design in this order. Each layer constrains the next.

| Layer | Question | Gets Wrong → |
|---|---|---|
| 1. Action Space | What tools does the agent have? | Agent picks wrong tool, or can't express the right action |
| 2. Tool Boundaries | What does each tool do and not do? | Tool does too much (uncontrollable) or too little (agent needs 5 calls for 1 task) |
| 3. Observation Format | What does the agent see after acting? | Agent misses critical info, or drowns in irrelevant detail |
| 4. Evaluation Signal | How do you know the agent is behaving well? | Silent failures, no feedback loop, can't diagnose regressions |
| 5. Harness Iteration | How do you improve based on agent behavior? | Random prompt patches instead of structural fixes |

---

## Layer 1: Action Space Design

The action space is the set of tools available to the agent. This is the single highest-leverage design decision.

### The Goldilocks Problem

| Too few tools | Too many tools |
|---|---|
| Agent can't express the right action | Agent wastes turns picking between similar tools |
| Forces creative misuse of existing tools | Increases probability of choosing the wrong tool |
| Feels constrained but is predictable | Feels flexible but is unpredictable |

### Design Rules

**Start minimal.** Give the agent only the tools it needs for the core task. Add tools only when you observe the agent failing because it cannot express a needed action.

**One tool per intent.** If two tools serve the same intent in different ways, keep only one — the one with clearer boundaries. An agent choosing between `search_documents` and `query_knowledge_base` will pick inconsistently.

**Name tools as verbs + objects.** The tool name is the strongest signal the LLM has for when to use it.

```
# Bad — ambiguous intent
"process_data"
"handle_request"
"run_operation"

# Good — clear intent
"search_customer_by_email"
"generate_sql_from_question"
"validate_sql_syntax"
```

**Group related operations.** If two operations always happen together, consider combining them into one tool. If they sometimes happen independently, keep them separate.

```
# Separate — sometimes you validate without executing
validate_sql(query) → {is_valid, errors}
execute_sql(query) → {rows, row_count}

# Combined — if validation always precedes execution in your system
execute_validated_sql(query) → {is_valid, errors, rows}
```

### Action Space Audit

For each tool in your action space, answer:

```
Tool: [name]
When should the agent use this? [one sentence]
When should the agent NOT use this? [one sentence]
What happens if the agent uses this at the wrong time? [consequence]
Is there another tool that overlaps with this? [which one, and how to disambiguate]
```

If you cannot answer these clearly, the tool boundary is wrong.

---

## Layer 2: Tool Boundary Design

Each tool needs a contract: what it accepts, what it returns, and what it does NOT do.

### Input Contract

**Be strict.** Accept exactly what is needed — no optional fields that change behavior dramatically, no overloaded parameters that mean different things in different contexts.

```
# Bad — overloaded parameter
search(query, mode="semantic|keyword|hybrid", filters=None, limit=None, offset=None, rerank=True, ...)

# Good — separate tools with focused inputs
search_semantic(query, limit=10) → results
search_keyword(query, filters, limit=10) → results
```

**Validate inputs before the LLM sees the error.** If a tool will fail on empty input, bad types, or out-of-range values, catch it at the boundary with a clear message — not a stack trace.

```python
def execute_sql(query: str) -> dict:
    if not query.strip():
        return {"error": "Empty query. Provide a SQL SELECT statement."}
    if not query.strip().upper().startswith("SELECT"):
        return {"error": f"Only SELECT queries allowed. Got: {query[:20]}..."}
    # proceed with execution
```

### Output Contract

**Return structured data, not prose.** The next reasoning step consumes your tool's output. Structured data is parseable; prose requires interpretation (which introduces variance).

```
# Bad — agent must parse prose
"Found 3 customers matching 'Acme'. The first one is Acme Corp (ID: 42) 
which was created in 2023. There's also Acme LLC..."

# Good — structured, scannable
{"results": [{"name": "Acme Corp", "id": 42, "created": "2023-01-15"}, ...], "total": 3}
```

**Include metadata the agent needs to decide what to do next.** Row count, truncation flag, error type — these enable deterministic routing.

```python
return {
    "rows": rows[:limit],
    "total_count": total,
    "truncated": total > limit,  # agent knows it's missing data
    "query_ms": elapsed_ms        # agent can flag slow queries
}
```

**Cap output size.** LLM context is finite. If a tool can return 10,000 rows, it will — and the agent will lose track of everything else.

```python
MAX_ROWS = 50
rows = cursor.fetchmany(MAX_ROWS + 1)
truncated = len(rows) > MAX_ROWS
return {"rows": rows[:MAX_ROWS], "truncated": truncated}
```

### The "Does NOT Do" List

For each tool, explicitly document what it does NOT handle. This prevents scope creep and makes debugging easier.

```
execute_sql:
  Does: Run a SELECT query, return rows
  Does NOT: Validate SQL syntax (that's validate_sql's job)
  Does NOT: Format results for display (that's the synthesizer's job)
  Does NOT: Retry on timeout (that's the orchestrator's job)
```

---

## Layer 3: Observation Format

After the agent acts, what does it see? This determines whether it can reason correctly about what to do next.

### The Signal-to-Noise Problem

The LLM reads the observation the same way it reads any text — linearly, with attention that decays. If the critical signal is buried in verbose output, the agent will miss it.

**Put the verdict first.** Status, error type, or key metric should be the first thing in the observation.

```
# Bad — buried signal
{"data": [... 200 rows ...], "metadata": {"query_time": "2.3s"}, "status": "partial"}

# Good — signal first
{"status": "partial", "row_count": 200, "truncated": true, "data": [... rows ...]}
```

**Summarize before details.** If the full output is large, lead with a summary the agent can reason about, followed by details it can scan if needed.

```python
def format_observation(results):
    summary = {
        "status": "success" if results else "no_results",
        "count": len(results),
        "columns": list(results[0].keys()) if results else [],
    }
    return {"summary": summary, "rows": results}
```

### What to Include vs Exclude

| Include | Exclude |
|---|---|
| Status (success / error / partial) | Stack traces (log them, don't show to agent) |
| Row count, total count | Raw database cursor metadata |
| Truncation flag | Internal timing breakdowns |
| Error type + actionable message | Full SQL query echo (agent already knows what it sent) |
| Key fields the agent needs for next decision | Fields only relevant to downstream display |

### Error Observations

When a tool fails, the observation must tell the agent **what went wrong** and **what to do about it**.

```
# Bad — agent has no idea what to do
{"error": "OperationalError: relation 'users' does not exist"}

# Good — actionable
{"error": "table_not_found", "table": "users", "suggestion": "Available tables: customers, accounts, sessions. Did you mean 'customers'?"}
```

**Categorize errors.** The agent needs to know: is this retryable? Should it try a different approach? Or should it give up?

```python
ERROR_CATEGORIES = {
    "syntax_error": "Fix the query syntax and retry",
    "table_not_found": "Check available tables and retry with correct name",
    "timeout": "Query too slow. Add filters or reduce scope",
    "permission_denied": "This operation is not allowed. Try a different approach",
    "rate_limit": "Wait and retry (transient)",
}
```

---

## Layer 4: Evaluation Signal Design

How do you know the harness is working? You need signals at two levels: **per-action** (is each tool call reasonable?) and **per-task** (did the agent accomplish the goal?).

### Per-Action Signals

Track these for every tool call:

```python
action_log = {
    "tool": tool_name,
    "input": tool_input,
    "output_summary": summarize(output),  # not full output — too expensive to store
    "latency_ms": elapsed,
    "success": not is_error,
    "error_type": error_category if is_error else None,
}
```

**Red flags in action logs:**
- Same tool called 3+ times with similar input → agent is stuck in a loop
- Tool A called immediately after Tool A fails → agent not adapting strategy
- Tool calls that don't use information from previous observations → agent ignoring context
- Steady increase in output token count per turn → context pollution

### Per-Task Signals

Define what "success" looks like before the agent runs:

```
Task: Answer user question from database
Success: Final answer cites specific data from query results
Partial: Answer is reasonable but missing data the DB has
Failure: Answer contradicts data, hallucinates, or gives up without trying
```

### Building an Evaluation Loop

```
1. Run agent on N test cases
2. Log all action sequences
3. For each case, score: success / partial / failure
4. For failures, identify: which action went wrong? which layer caused it?
   - Wrong tool chosen → Layer 1 (action space)
   - Right tool, wrong input → Layer 2 (tool boundary)
   - Right output, agent misread it → Layer 3 (observation format)
   - Agent couldn't tell it was failing → Layer 4 (evaluation signal)
5. Fix the identified layer, not the prompt
```

---

## Layer 5: Harness Iteration

When the agent misbehaves, diagnose which layer to fix.

### The Diagnosis Ladder

Go through in order. Stop at the first match.

```
1. Is the agent choosing the wrong tool?
   → Layer 1: Action space is ambiguous. Rename, merge, or remove tools.

2. Is the agent choosing the right tool but with wrong inputs?
   → Layer 2: Tool boundary is unclear. Tighten input contract, 
     add validation, improve error messages.

3. Is the agent getting the right output but making wrong conclusions?
   → Layer 3: Observation format is misleading. Restructure output, 
     put signal first, remove noise.

4. Is the agent failing silently (no one notices until too late)?
   → Layer 4: Evaluation signals are missing. Add logging, define 
     success criteria, build automated checks.

5. None of the above — agent has the right tools, right observations, 
   but still reasons incorrectly?
   → NOW it's a prompt problem. Only now.
```

**The most common mistake:** Jumping to step 5 (prompt fix) before checking steps 1-4. Prompt patches on harness problems create fragile systems that break on the next edge case.

### Harness Change Protocol

When you change the harness, follow the same experiment-driven-development process:

1. **Evidence:** What behavior did you observe? (action logs, not assumptions)
2. **Diagnosis:** Which layer is responsible? (use the ladder above)
3. **Change:** Smallest modification to that layer
4. **Validation:** Re-run the failing cases + positive controls
5. **Documentation:** Record what changed and why

See skill: `experiment-driven-development` for the full process.

---

## Known Failure Modes

| Failure | Symptom | Root Cause | Fix |
|---|---|---|---|
| Tool confusion | Agent alternates between 2 tools for the same task | Overlapping tools in action space | Merge or clearly differentiate tools (Layer 1) |
| Creative misuse | Agent uses tool X for something tool Y should do | Missing tool in action space, or tool X's name is too generic | Add the missing tool or rename (Layer 1) |
| Input guessing | Agent passes malformed inputs that fail validation | Tool input contract is unclear or undocumented | Tighten input validation with clear error messages (Layer 2) |
| Observation blindness | Agent ignores important data in tool output | Critical info buried in verbose output | Restructure: signal first, summary before detail (Layer 3) |
| Silent failure | Agent produces wrong answer, nobody notices | No per-task evaluation signal | Define success criteria, add automated checks (Layer 4) |
| Prompt patching | Prompt grows with edge-case rules, quality degrades | Structural harness problem addressed with prompt rules | Use diagnosis ladder, fix the right layer (Layer 5) |
| Context pollution | Agent performance degrades over long sessions | Tool outputs too large, filling context window | Cap output size, summarize, exclude unnecessary fields (Layer 3) |

---

## Harness Design Checklist

### Before building
- [ ] Core task defined — what the agent should accomplish
- [ ] Action space drafted — minimal set of tools for the task
- [ ] Each tool has a one-sentence "when to use" and "when NOT to use"
- [ ] No two tools serve the same intent

### For each tool
- [ ] Input contract: strict types, validated, clear error on bad input
- [ ] Output contract: structured data, not prose
- [ ] Output capped: maximum size enforced
- [ ] Error format: category + actionable message
- [ ] "Does NOT do" list documented

### For observations
- [ ] Status/verdict is the first field in every observation
- [ ] Summary precedes detail in large outputs
- [ ] Error observations tell the agent what to do next
- [ ] No raw stack traces or internal metadata exposed to agent

### For evaluation
- [ ] Per-action logging in place (tool, input summary, success, latency)
- [ ] Per-task success criteria defined
- [ ] Red flag detection: loops, repeated failures, context growth
- [ ] Evaluation loop: test cases → score → diagnose layer → fix

---

## Key Principles

**Harness before prompt.** Design what the agent can do and see before writing how it should think. The prompt adapts to the harness, not the other way around.

**One tool, one intent.** Ambiguous action spaces cause the majority of wrong-tool-choice failures. When in doubt, remove a tool rather than add a rule about when to use it.

**Signal first.** The LLM reads observations linearly. If the most important information isn't first, it may be ignored.

**Diagnose the layer, not the symptom.** When the agent misbehaves, use the diagnosis ladder to find which harness layer is responsible. Fix that layer. Prompt patches on harness problems are fragile.

**Cap everything.** Output size, retry count, context growth. Uncapped outputs are the leading cause of context pollution and agent performance degradation over long sessions.
