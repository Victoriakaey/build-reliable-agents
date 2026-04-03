---
name: critic-judge-design
description: "Use when designing any LLM-as-Judge, Critic, or Evaluator node. Covers input structure, output schema, chain-of-thought ordering, single-pass vs multi-stage tradeoffs, and known failure modes. Prevents the most common design mistakes that cause Critic nodes to be unreliable."
---

# Critic / Judge Node Design

> **Terminology:** This skill uses "Critic", "Judge", and "Evaluator" interchangeably. Pick one term for your codebase and use it consistently.

## The Core Problem

A Critic node has one primary job: judge whether something meets a quality bar. Everything else it does (generating feedback, planning follow-ups, classifying failure types) is secondary.

**The most common mistake:** making the Critic do too much in one pass. When a node simultaneously judges quality AND generates structured follow-up plans, the two tasks interfere with each other — and the primary job (judgment) suffers.

---

## Step 1: Define the Primary Job

Before designing any Critic, answer:

1. **What exactly is being judged?** (evidence sufficiency, answer quality, SQL correctness, etc.)
2. **What is the binary output?** (sufficient/insufficient, pass/fail, correct/incorrect)
3. **What does the Critic need to see to make this judgment?**
4. **What does NOT need to be in the Critic's input?**

The cleaner the primary job definition, the more reliable the Critic.

---

## Step 2: Design the Input Structure

**This is the most important decision.** Input structure determines reasoning patterns more than prompt instructions do.

### The Organization Problem

If you organize input by sub-question:
```
SQ1: "Find the patch window" → 0 rows
SQ2: "Find rollback artifacts" → 3 rows (contains patch window)
```

The LLM will evaluate each sub-question against its own text. It will conclude "patch window not found" from SQ1 even if SQ2's rows contain the answer. **The input structure primes per-sub-question reasoning.**

### Flat Evidence Pool (preferred for sufficiency judgment)

Instead of organizing by sub-question, present evidence as a flat pool:
```
Original question: ...
Retrieved evidence:
- Row 1: {customer: "X", patch_window: "2026-03-24 02:00", ...}
- Row 2: {customer: "X", rollback_cmd: "orchestrator rollback ...", ...}
```

This forces the LLM to evaluate evidence against the original question, not against sub-question text.

**Tradeoff:** You lose sub-question provenance (can't say "SQ2 found this"). Gain: correct holistic reasoning.

### When to keep sub-question organization

Keep it when the Critic's job IS per-sub-question evaluation (e.g., "did this specific sub-question return useful data?"). Remove it when the Critic's job is holistic sufficiency (e.g., "can the original question be answered from all evidence combined?").

---

## Step 3: Design the Output Schema

**Chain-of-thought ordering matters.** The LLM reasons in the order it writes. Put the reasoning BEFORE the judgment, not after.

### Wrong order (judgment first):
```json
{
  "is_sufficient": true,
  "judgment": "Evidence covers the patch window...",
  "confidence": 90
}
```

The LLM commits to `is_sufficient` before writing the reasoning. The reasoning becomes post-hoc justification.

### Right order (reasoning first):
```json
{
  "evidence_present": ["patch_window", "rollback_cmd"],
  "evidence_missing": [],
  "judgment": "Both required fields are present in the retrieved rows...",
  "confidence": 90,
  "is_sufficient": true
}
```

The LLM must enumerate what's present and missing before committing to a verdict. This produces more accurate judgments.

### Coverage score vs binary

Use an intermediate score (1-4) and derive a binary from it:
```
coverage_score: 1-4
is_sufficient: derived from coverage_score >= 3
```

This gives the LLM room to express nuance, and keeps the routing logic deterministic (binary derived from score, not LLM-generated bool).

---

## Step 4: Decide Single-Pass vs Multi-Stage

### Single-pass Critic (judgment + feedback in one call)

**Use when:**
- Feedback is simple (just a list of missing sub-questions)
- Prompt stays under ~60 lines
- The feedback doesn't require complex reasoning beyond what judgment already does

**Risk:** As feedback complexity grows, judgment quality degrades. The two tasks interfere.

### Multi-stage Critic (judgment in one call, feedback in another)

**Use when:**
- Feedback requires detailed planning (gap types, dependency hints, suggested tables, strategy notes)
- Prompt is already long (>80 lines)
- Judgment quality is inconsistent

**Structure:**
```
Stage 1 — Judgment node:
  Input: original question + flat evidence pool
  Output: is_sufficient, retrieval_outcome, confidence, brief judgment

Stage 2 — Feedback Planner node (only runs if insufficient):
  Input: original question + evidence + judgment from Stage 1
  Output: gap_type, missing_evidence, suggested_tables, dependency_hints

Stage 3 — Follow-up Parser (already exists):
  Input: feedback from Stage 2
  Output: new sub-questions
```

**Cost:** One extra LLM call on insufficient cases only. No cost on sufficient cases.

---

## Step 5: Routing Logic

**Always derive routing from deterministic fields, not LLM-generated booleans.**

```python
# Fragile — LLM generates the bool
if state["is_sufficient"]:
    route to synthesizer

# Robust — derive from score
if state["coverage_score"] >= 3:
    route to synthesizer
```

For multi-outcome routing (sufficient / needs_more / no_evidence / unsupported_premise):
```python
match state["retrieval_outcome"]:
    case "sufficient": route to synthesizer
    case "needs_more_retrieval": route to follow-up parser (if rounds < max)
    case "no_evidence": route directly to synthesizer
    case "unsupported_premise": route directly to synthesizer
    case _: route to synthesizer (safe default)
```

Always have a safe default that prevents infinite loops.

---

## Step 6: Loop Guard

Every Critic-driven loop MUST have a hard exit condition independent of Critic judgment:

```python
if critic_round_count >= max_rounds:
    force route to synthesizer
    # regardless of what Critic says
```

Also add sub-question deduplication to prevent the Critic from requesting the same sub-question repeatedly:
```python
seen_signatures = set()
# before adding to pending queue, check signature
if sub_question_signature not in seen_signatures:
    pending_queue.append(sub_question)
    seen_signatures.add(sub_question_signature)
```

---

## Known Failure Modes

| Failure | Symptom | Root Cause | Fix |
|---|---|---|---|
| Per-sub-question reasoning | Critic judges each SQ independently, misses cross-SQ evidence | Input organized by sub-question | Flatten evidence pool |
| Post-hoc justification | Critic commits to verdict, writes reasoning to match | Judgment field before reasoning in schema | Reorder: reasoning → score → verdict |
| Ghost rules | Critical rules ignored, less important rules followed | Prompt too long, rules competing for attention | Simplify prompt, put critical rules first |
| Infinite loops | Critic keeps saying insufficient even when evidence exists | No loop guard, or loop guard too permissive | Hard exit at max_rounds regardless of Critic |
| Follow-up drift | Follow-up sub-questions drift from original question scope | Critic generates follow-ups without entity anchoring | Require Critic to scan completed SQs for entity names before generating feedback |