---
name: problem-exploration
description: "Use when facing a technical problem with multiple possible solutions, or when a previous approach failed and you need to reason about alternatives. Structures the space of options before committing to any one. Core output: a brainstorm log with options, tradeoffs, a chosen approach, and unresolved gaps that must be answered before implementation starts. Use before experiment-driven-development Step 4 when the design space is large or uncertain."
---

# Problem Exploration

## The Core Problem

When facing a hard technical problem, the instinct is to jump to the first solution that seems reasonable and start implementing. This is almost always wrong. The first solution is rarely the best one, and unexamined alternatives often contain the insight that makes the chosen approach actually work.

**The discipline:** Before committing to any approach, exhaust the solution space. The goal is not to find the perfect solution — it's to make a well-informed choice with known tradeoffs and known gaps.

---

## When to Use This

Use this skill when:
- A previous approach failed and you need to reconsider
- There are multiple plausible solutions and you're not sure which to pick
- A problem has constraints that eliminate obvious solutions
- You need to record why you chose X over Y for future reference
- You're about to make a decision that is hard to reverse

Do NOT use this for:
- Straightforward fixes with one obvious correct solution
- Tasks where the approach is already decided and just needs execution

---

## The Output: A Brainstorm Log

Every problem exploration session produces a log file saved to:
```
docs/working-notes/<topic>-brainstorm-log.md
```

The log is the artifact. It captures the reasoning so future sessions don't repeat the same thinking.

---

## Step 1: Define the Problem Precisely

Before listing any solutions, write a precise problem statement:

```
Problem: [One sentence — what is broken or missing]
Current behavior: [What the system does now]
Desired behavior: [What it should do instead]
Constraints: [What solutions must not violate — performance, cost, 
              architecture boundaries, backward compatibility, etc.]
Root cause hypothesis: [Why does the problem exist — which layer, 
                        which assumption was wrong]
```

**The constraint list is critical.** Many options get eliminated immediately once constraints are explicit. Write them before listing options, not after.

---

## Step 2: Generate Options Exhaustively

List ALL plausible approaches before evaluating any of them. Do not evaluate while listing — this prematurely closes the solution space.

For each option:
```
### Option [Letter]: [Short name]

What it does: [One paragraph — mechanism, not benefits]
```

Aim for 3–6 options. Fewer than 3 means you haven't explored enough; more than 6 usually means some options are variants of each other — merge them. Common sources of missed options:
- The "do nothing / accept limitation" option (always valid, often underconsidered)
- The "move the problem to a different layer" option
- The "make it deterministic instead of LLM-driven" option
- The "split into two simpler problems" option

---

## Step 3: Evaluate Each Option

For each option, fill in a tradeoff table:

```
| Dimension          | Assessment |
|--------------------|------------|
| Files changed      | [N files, which ones] |
| New components     | [Yes/No — new nodes, new files, new dependencies] |
| LLM calls added    | [+N per query, or zero] |
| Latency impact     | [Minimal / +Xs per query / significant] |
| Easy query risk    | [Zero / Low / Medium / High] |
| Effectiveness      | [Low / Medium / High — and why] |
| Determinism        | [Low / Medium / High] |
| Reversibility      | [Easy to revert / Hard to revert] |
| Implementation     | [Trivial / Simple / Medium / Complex] |
```

Then write:
```
Pro: [What this option does better than alternatives]
Con: [What this option trades away]
Rejected because / Chosen because: [One clear sentence]
```

---

## Step 4: Choose and Justify

After evaluating all options:

```
Decision: Option [X]

Why chosen:
- [Primary reason]
- [Secondary reason]
- [What made the alternatives less suitable]

What we are explicitly accepting:
- [Known limitation 1 — and why it's acceptable]
- [Known limitation 2 — and why it's acceptable]
```

The "what we are explicitly accepting" section is important. Every choice has a cost. Naming it prevents future confusion about why the system behaves a certain way.

---

## Step 5: Identify Unresolved Gaps

**This is the most important step.** Before any implementation starts, enumerate everything that is not yet figured out.

For each gap:
```
### Gap N: [Short name]

Problem: [What is not yet resolved]
Options considered: [If any]
Blocking: [Does this gap block implementation, or can it be resolved during implementation]
```

**A gap is a blocker if:**
- The implementation approach changes depending on how the gap is resolved
- Getting the gap wrong requires redoing work already done

**Do not start implementation if a blocking gap exists.** Resolve the gap first, or explicitly decide to accept the uncertainty and document it.

In practice, gaps discovered after coding starts are significantly more expensive than gaps discovered during exploration — often requiring rework of already-completed implementation.

---

## Step 6: Define Validation Criteria

Before closing the brainstorm, write down how you will know the chosen approach worked:

```
Success criteria:
- [Case X should now produce result Y]
- [Previously-stable cases must remain unaffected]
- [Performance metric: N LLM calls, M seconds]

Regression canaries:
- [Case A — must still pass]
- [Case B — must still pass]

If these canaries fail after implementation: revert immediately.
```

---

## Brainstorm Log Template

```markdown
# [Topic] Brainstorm Log

## Problem Statement
Problem: 
Current behavior: 
Desired behavior: 
Constraints: 
Root cause hypothesis: 

---

## Options

### Option A: [Name]
What it does: 

| Dimension | Assessment |
|---|---|
| Files changed | |
| New components | |
| LLM calls added | |
| Latency impact | |
| Easy query risk | |
| Effectiveness | |
| Determinism | |
| Reversibility | |
| Implementation | |

Pro: 
Con: 
Status: REJECTED — [reason] / CHOSEN — [reason]

### Option B: [Name]
[same structure]

---

## Decision

Decision: Option [X]

Why chosen:
- 
- 

What we are explicitly accepting:
- 

---

## Unresolved Gaps

### Gap 1: [Name]
Problem: 
Blocking: Yes / No
Resolution needed before: [before coding / during coding / after first test run]

---

## Validation Criteria

Success criteria:
- 

Regression canaries:
- 
```

---

## Key Principles

**Exhaust before evaluating.** List all options before judging any of them. Premature evaluation closes the solution space.

**Constraints first.** Write constraints before options. Many options eliminate themselves.

**Name what you're accepting.** Every choice has a cost. If you don't name it, you'll be surprised by it later.

**Gaps are blockers.** An unresolved gap that changes the implementation approach must be resolved before coding starts. A gap discovered after coding is a rewrite.

**The log is the artifact.** The value of this process is the written record — not the decision itself. Future sessions should not repeat the same reasoning.