---
name: prompt-change-management
description: "Use before making any prompt change in an LLM system. Covers the full cycle: pre-change documentation, change scoping, post-change validation, and rollback protocol. Prevents silent regressions caused by prompt changes with unexpected downstream effects."
---

# Prompt Change Management

## The Core Problem

Prompt changes have blast radius. A change to Parser prompt affects sub-question phrasing, which affects what Critic reads, which affects sufficiency judgment — even though you only touched one file. Silent regressions are the norm, not the exception.

**Rule: Never commit a prompt change without a validation run.**

---

## Step 0: Before You Touch Anything

Answer these questions first. If you can't answer them, do not proceed.

1. **What behavior are you trying to change?** (specific, observable)
2. **What is the current behavior?** (with a concrete example from logs)
3. **Which node's prompt are you changing?**
4. **Which other nodes read output from that node?** (these are your blast radius)
5. **What test cases will you run to confirm the change worked?**
6. **What test cases will you run to confirm nothing else broke?**

Write these down before opening any prompt file.

---

## Step 1: Document the Change Before Committing

Every prompt change MUST have a log entry in `docs/prompt-change-log.md` BEFORE the commit. Format:

```
| # | Commit | Task | What Changed | Why | Validation Log |
|---|--------|------|-------------|-----|---------------|
| P7 | pending | [task name] | [exact diff in plain language] | [root cause this fixes] | pending |
```

Fill in commit hash and validation log path after the run completes.

---

## Step 2: Scope the Change

**Prefer the smallest possible change that fixes the problem.**

Before writing new prompt text, ask:
- Is this a prompt problem or an input structure problem? (Input structure problems cannot be fixed by prompt changes — see `critic-judge-design` skill)
- Can a few-shot example fix this instead of a new rule?
- Will adding this rule conflict with any existing rule in the same prompt?
- Is this prompt already too long? (>100 lines is a warning sign — rules start competing for attention)

**If the prompt is already long:** consider whether the new rule is truly necessary, or whether it's patching a structural problem that needs a different fix.

---

## Step 3: Identify Downstream Effects

For each node whose prompt you're changing, identify what downstream nodes consume its output:

```
Parser prompt change
  → affects sub_question text
    → affects Generator SQL strategy
    → affects Critic sufficiency judgment (Critic reads sub_question text)
      → affects whether follow-up rounds trigger
        → affects Synthesizer input

Generator prompt change
  → affects SQL quality
    → affects Executor row counts
      → affects Critic coverage assessment

Critic prompt change
  → affects retrieval_outcome routing
  → affects feedback object quality
    → affects follow-up Parser decomposition
```

**Known high-risk pattern:** Parser prompt examples that use entity names from your actual test data will prime the LLM to copy those phrasings, causing consistent sub-question phrasing shifts that propagate to Critic judgment.

---

## Step 4: Run Validation

### Minimum validation required for any prompt change:

```bash
python terminal_app.py \
  --batch-file test-inputs/critic/core.txt \
  --logs-dir logs/<change-name>/<timestamp> \
  --max-critic-rounds 3 \
  --sleep-seconds 5 \
  --retry-cooldown-seconds 30
```

Save a `_metadata.txt` in the log directory:
```
commit: <hash> <message>
command: <full command>
batch_file: <path>
max_critic_rounds: <N>
model: <model name>
temperature: <N>
seed: <N>
started_at: <ISO timestamp>
results: <N/M sufficient>
notes: <what you changed and why>
```

### Controlled comparison rules:

When comparing before/after runs, these MUST be identical:
- Batch file (same cases, same order)
- `--max-critic-rounds`
- All other CLI parameters
- Model, temperature, seed

If any parameter differs, the comparison is not valid for causal claims.

---

## Step 5: Evaluate Results

After the run, check:

1. **Did the target behavior change?** (the thing you were trying to fix)
2. **Did any previously-passing cases regress?**
3. **Is the result stable?** (run twice — if results differ significantly, the change introduced instability)

**Passing threshold:** All previously-stable cases must still pass. If any regression appears, the change is not safe to ship regardless of whether it fixed the target behavior.

---

## Step 6: Rollback Protocol

If a regression is detected:

1. `git revert <commit>` immediately
2. Document the regression in the prompt change log with the validation log path
3. Analyze root cause before attempting the fix again (see `regression-testing` skill for bisect protocol)

**Do not attempt to fix the regression with another prompt change.** Stacked prompt changes make root cause analysis impossible.

---

## Known High-Risk Prompt Patterns

| Pattern | Risk | Mitigation |
|---|---|---|
| Entity-specific examples in prompt | LLM copies entity names into outputs, causing phrasing drift | Use generic examples only |
| Rules added after 100+ line prompt | Rules compete for attention, critical rules get ignored | Simplify or split the prompt |
| Same node doing judgment + planning | Reasoning quality degrades when doing two conflicting things at once | Split into two nodes |
| Fixing a structural problem with a prompt rule | Prompt says one thing, data structure says another — data structure wins | Fix the structure, not the prompt |