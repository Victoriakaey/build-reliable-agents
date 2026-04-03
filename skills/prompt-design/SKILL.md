---
name: prompt-design
description: "Use before writing any new LLM component prompt, or when an existing prompt is producing unstable, inconsistent, or wrong outputs. Two modes: (1) design from scratch by defining component boundaries and structure before writing, (2) diagnose an existing prompt to find root cause of failure. Covers all dimensions of prompt quality: job definition, input structure, output schema, rules, few-shot examples, system/user split, token budget, and prompt decay over time."
---

# Prompt Design

## The Core Problem

Most prompt failures are not wording problems. They are structural problems — the component was asked to do too many things, the input organized the wrong way, the schema committed to a verdict before reasoning, or the prompt grew rules over time until it contradicted itself.

**The discipline:** Define the structure before writing the words. Diagnose the structure before adding rules.

---

## Two Modes

**Mode 1: Design From Scratch** — writing a new prompt for a new node → Start at Step 1

**Mode 2: Diagnose an Existing Prompt** — prompt is producing wrong or unstable outputs → Start at Step D1

---

# Mode 1: Design From Scratch

## Step 1: Define the Node's Single Primary Job

Before opening any file, complete this sentence:

**"This node takes [input] and produces [output] so that [downstream consumer] can [do what]."**

If you cannot write this without "and" connecting two different jobs, the component is doing too much. Split it before writing any prompt.

Examples:
- "This node takes a user query and produces a list of sub-questions so that the Generator can write targeted SQL queries."
- "This node takes retrieved evidence and produces a sufficiency judgment so that the router can decide whether to loop or synthesize."

**What the component must NOT do** — list explicitly:
```
This node does NOT:
- [ ] [job that belongs to another node]
- [ ] [job that belongs to another node]
```

If you want to add something here mid-prompt-writing, stop. You are about to add a second job.

---

## Step 2: Define Success Criteria Before Writing

Write 2-3 concrete examples before writing any prompt text:

```
Input:  [example]
Output: [correct output]
Why:    [what makes this correct]

Failure case:
Input:        [input that should produce a specific output]
Wrong output: [what a bad prompt produces]
Right output: [what the right prompt produces]
Why it fails: [which structural problem caused it]
```

Writing failure cases forces you to find the structural problem before it finds you.

---

## Step 3: Design the Input Structure

**Input structure determines reasoning patterns more than prompt instructions do.**

When the LLM sees input organized one way, it reasons in that shape — regardless of what the prompt says. When prompt instructions and input organization conflict, input organization wins.

```
// Input organized by sub-question → primes per-sub-question reasoning
SQ1: "Find patch window" → 0 rows
SQ2: "Find rollback info" → 3 rows  ← contains patch window

// LLM reads SQ1, sees 0 rows, concludes "patch window not found"
// even though SQ2's rows contain the answer
// Prompt says "cross-reference all evidence" — input structure says otherwise
// Input structure wins.
```

**Ask before designing input:**
- What reasoning pattern do I need? (holistic / per-item / sequential)
- Does my input organization prime that pattern, or a different one?
- Am I including fields the component does not need? (unnecessary fields invite the LLM to reason about them)

**Rule:** If you need holistic reasoning across all evidence, present evidence as a flat pool — not organized by sub-question, case, or category.

---

## Step 4: Decide System Prompt vs User Prompt

Not everything belongs in the same place. This split affects instruction-following quality.

| Put in System Prompt | Put in User Prompt |
|---|---|
| Node role definition | Per-request input (current query, current evidence) |
| Hard rules that never change | Context that changes per call |
| Output schema definition | Dynamic fields (current retry count, current round) |
| Few-shot examples | Results from upstream nodes |
| Static schema / reference data | |

**Why it matters:** System prompt content is treated as persistent instruction. User prompt content is treated as current context. Mixing them causes the LLM to treat hard rules as negotiable context, or to ignore dynamic context because it is buried in static instructions.

---

## Step 5: Design the Output Schema

### Rule 1: Reasoning before verdict

The LLM reasons in the order it writes. Verdict first means reasoning becomes post-hoc justification.

```json
// WRONG — verdict first
{
  "is_sufficient": true,
  "judgment": "Evidence covers the required fields..."
}

// RIGHT — reasoning first, verdict last
{
  "evidence_present": ["patch_window", "rollback_cmd"],
  "evidence_missing": [],
  "judgment": "Both required fields are present in the retrieved rows...",
  "coverage_score": 4,
  "is_sufficient": true
}
```

### Rule 2: Derive booleans from scores

```json
// Less stable — LLM generates boolean directly
"is_sufficient": true

// More stable — LLM generates score, boolean is derived in code
"coverage_score": 3,   // LLM generates this
"is_sufficient": true  // derived: coverage_score >= 3
```

### Rule 3: One schema, one job

If your schema has fields from two different reasoning tasks, split the component.

```json
// Too much — judgment AND follow-up planning in one schema
{
  "is_sufficient": false,
  "judgment": "...",
  "gap_type": "missing_evidence",           // follow-up planning
  "suggested_target_tables": ["artifacts"], // follow-up planning
  "dependency_hints": ["use SQ1 for SQ2"]  // follow-up planning
}

// Right — split into two components
// Node 1 (Critic): judgment only
{ "coverage_score": 2, "judgment": "...", "retrieval_outcome": "needs_more_retrieval" }

// Node 2 (Feedback Planner): planning only, runs after Node 1 if insufficient
{ "gap_type": "...", "suggested_target_tables": [...], "dependency_hints": [...] }
```

### Rule 4: Constrain routing-critical fields to enums

If a field drives downstream routing, it must be an enum — not free-form text.

```json
// Fragile
"retrieval_outcome": "I think there is not enough evidence"

// Robust
"retrieval_outcome": "needs_more_retrieval"
// constrained to: sufficient / needs_more_retrieval / no_evidence / unsupported_premise
```

---

## Step 6: Write the Rules

### Structure

```
[Most critical rule — if this breaks, everything else is wrong]
[Second most critical rule]
...
[Edge case rules]
[Formatting rules — always last]
```

### Rules for rules

**Maximum 5-7 rules.** Every rule added dilutes every other rule. In practice, prompts with 15+ rules show significantly degraded per-rule adherence — the LLM follows a subset consistently and ignores the rest unpredictably.

**Most critical rule goes first.** Attention degrades toward the end of long prompts. The rule you most need followed must be at the top.

**One rule = one behavior.** If a rule contains "and", split it into two rules.

**Never use entity-specific examples in rules.** Examples using real names from your test data prime the LLM to copy those names into outputs, causing phrasing drift that propagates through the pipeline.

```
// BAD — entity-specific
"For example, if asked about Verdant Bay's patch window, prefer source_type=structured"

// GOOD — generic
"For example, if asked about a customer's scheduled maintenance window, prefer source_type=structured"
```

**If a rule is patching a structural problem, fix the structure instead.** A rule that says "make sure to cross-reference evidence across all sub-questions" is patching an input organization problem. Fix the input organization — the rule will not reliably work.

---

## Step 7: Write Few-Shot Examples

Few-shot examples are the highest-leverage part of a prompt. One good example teaches more than five rules.

**Include both bad and good examples.** Only showing correct outputs leaves the LLM without a reference for what to avoid.

```
BAD example:
Input:  [input]
Output: { "field": "wrong value" }
Why wrong: [explicit explanation]

GOOD example:
Input:  [same input]
Output: { "field": "correct value" }
Why right: [explicit explanation]
```

**2-3 examples is usually enough.** More than 5 inflates token usage without proportional quality gain. If you need more than 5 examples to cover the cases, the component's job definition is probably too broad.

**All examples must be generic.** No real customer names, no real dates from your dataset, no real field names from a specific deployment. Use placeholders.

---

## Step 8: Token Budget Check

| Prompt Size | Signal |
|---|---|
| < 1K tokens | Good — focused |
| 1K–3K tokens | Acceptable |
| 3K–5K tokens | Warning — check for scope creep |
| > 5K tokens | Problem — likely doing too much |

**Token size is an engineering constraint, not just a quality concern.** Depending on your rate limit tier, a 7K-token prompt can severely limit parallelism (e.g., at 30K TPM, only ~4 concurrent calls). Large prompts increase cost, reduce batch throughput, and constrain how many validation runs you can afford.

If over 5K tokens, go back to Step 1 and ask whether the component is doing too much.

---

## Step 9: Different-Context Check

If this component is called in multiple contexts (initial call vs follow-up, different upstream inputs), ask: does the same prompt work correctly in all contexts?

If contexts are genuinely different, use different prompt functions:

```python
# If one prompt works for all contexts
def build_parser_prompt(query, context): ...

# If contexts are genuinely different
def build_initial_parser_prompt(query): ...
def build_followup_parser_prompt(query, critic_feedback, seen_subquestions): ...
```

Using one prompt for genuinely different contexts causes stylistically inconsistent outputs across calls — which matters when downstream nodes read the output text as part of their reasoning input.

---

---

# Mode 2: Diagnose an Existing Prompt

## Step D1: Characterize the Failure Pattern

| Failure Pattern | Likely Root Cause |
|---|---|
| Correct on easy cases, wrong on hard cases | Rules too general, edge cases missing |
| Same input → different outputs across runs | Competing rules, or schema primes non-deterministic reasoning |
| Output technically correct but misses the point | Node job definition is wrong |
| Critical rule ignored on some inputs | Rule buried too deep, or competing with others |
| Reasoning looks right but verdict is wrong | Verdict before reasoning in schema |
| Node does X correctly but breaks Y | Two jobs in one prompt |
| Prompt worked before, degraded over time | Prompt decay — accumulated conflicting rules |
| Works in initial call, fails in follow-up call | Single prompt used for genuinely different contexts |

---

## Step D2: Check the Node Boundary

Apply the single-sentence test. Can you complete it without "and"?

**Real example — Critic node boundary failure:**

Wrong: "This node takes evidence and produces a sufficiency judgment AND generates structured feedback AND classifies the retrieval outcome type."

Three jobs. The primary job (judgment) gets crowded out by secondary jobs. Fix: split into Critic (judgment only) + Feedback Planner (planning only).

---

## Step D3: Check the Input Structure

Does the input organization prime the reasoning pattern the prompt requires?

- If the prompt asks for holistic reasoning but input is organized per-item → fix the input structure, not the prompt
- If the prompt asks the LLM to ignore certain fields → remove those fields from the input instead of adding a rule

---

## Step D4: Check System/User Split

Is static instruction mixed with dynamic context in the same prompt section? Move static content to system prompt, dynamic content to user prompt.

---

## Step D5: Check the Output Schema

1. Does reasoning come before verdict? If not → reorder fields
2. Are booleans derived from scores? If not → add intermediate score, derive in code
3. Does schema contain fields from two different jobs? If yes → split node
4. Are routing-critical fields free-form text? If yes → constrain to enum

---

## Step D6: Check the Rules

For each rule in the prompt, ask:
- Is this patching a structural problem? → Fix the structure instead
- Is it buried after 5+ other rules? → Move to top
- Does it conflict with another rule? → One is wrong, remove it
- Is it entity-specific? → Generalize
- Total rule count > 7? → Consider splitting the component

---

## Step D7: Check for Prompt Decay

Prompt decay happens when a clean prompt accumulates rules over time — each added to fix a specific edge case — until rules conflict and critical instructions get buried.

**Signs of decay:**
- Prompt has grown significantly since first written (check git blame or change log)
- Rules near the bottom were added reactively after failures
- You can find two rules that partially contradict each other
- Works on tested cases but fails on new cases in the same family

**Fix: audit and rewrite, not patch.**
Extract the 3-5 most critical rules. Rebuild from scratch with those as the foundation. Test before adding anything back. Patching a decayed prompt with more rules accelerates the decay.

---

## Diagnostic Summary

```
Root cause:       [boundary / input structure / schema / rules / system-user split /
                   token budget / context mismatch / prompt decay]
Evidence:         [specific quote from prompt or log that shows the problem]
Fix:              [which step to apply]
Expected outcome: [what should change after the fix]
Regression risk:  [which currently-passing cases might be affected]
```

---

## Key Principles

**Define before writing.** The prompt is a specification of a job. If the job is unclear, the specification will be wrong no matter how carefully it is worded.

**Structure beats instructions.** Input organization, output schema ordering, and system/user split shape reasoning more than any rule you write. Fix structure before adding rules.

**One job per node.** If a component does two things, split it. The extra LLM call is cheaper than the debugging cost of an overloaded prompt.

**Rules compete.** Every rule added dilutes every other rule. Fewer, more important rules at the top outperform many rules scattered throughout.

**Prompts decay.** A clean prompt accumulates rules over time. Audit periodically — rewrite when rules start conflicting rather than patching further.

**Never use entity-specific examples.** They prime phrasing drift that propagates through the pipeline.

**Token size is an engineering constraint.** Large prompts increase cost, reduce throughput, and constrain validation frequency.