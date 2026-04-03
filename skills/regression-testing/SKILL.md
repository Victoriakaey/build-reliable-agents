---
name: regression-testing
description: "Use when comparing two versions of an LLM system, debugging non-deterministic behavior, or establishing a baseline before making changes. Covers controlled comparison protocol, determinism verification, and bisect protocol for finding which change caused a regression."
---

# Regression Testing for LLM Systems

## The Core Problem

LLM systems have two sources of variance that look identical from the outside:
1. **Code/prompt changes** — something you did caused the behavior to change
2. **LLM non-determinism** — same code, same prompt, different output due to sampling variance

You cannot distinguish between these without a controlled protocol. Every "fix" that isn't measured is a guess.

---

## Step 0: Establish Determinism First

Before running any comparison, verify your system is as deterministic as possible.

**Check these in order:**

1. **Temperature = 0** — is it set in your LLM config?
2. **Seed parameter** — is a fixed seed set? (OpenAI: `seed=42`, best-effort but significantly reduces variance)
3. **No shared mutable state** — verify no module-level mutation, no shared checkpointer across runs
4. **No parallel runs** — sequential only; parallel runs share API quota and cause cascading rate limit errors

**Verify determinism before proceeding:**
Run the same batch file twice with identical parameters. If outputs differ significantly, you have a determinism problem that must be fixed before any comparison is meaningful.

```bash
# Run 1
python terminal_app.py --batch-file test-inputs/core.txt \
  --logs-dir logs/determinism-check/run1 --max-critic-rounds 3

# Run 2 (immediately after)
python terminal_app.py --batch-file test-inputs/core.txt \
  --logs-dir logs/determinism-check/run2 --max-critic-rounds 3

# Compare outputs manually
```

If runs differ: do not proceed. Fix determinism first (seed, temperature, state isolation).

---

## Step 1: Define Your Test Suite

A test suite must have:
- **Fixed cases** — same queries every time, in the same order
- **Known expected outputs** — what "correct" looks like for each case
- **Difficulty labels** — easy cases (should always pass) and hard cases (aspirational)
- **Minimum size** — at least 5-7 cases covering different query types

Easy cases are your regression canaries. If an easy case fails after a change, something is wrong regardless of whether the hard cases improved.

---

## Step 2: Establish a Baseline

Before making any change, run the full test suite and record:

```
baseline_commit: <hash>
date: <ISO date>
results: [case1: sufficient, case2: sufficient, case3: no_evidence, ...]
pass_rate: N/M
notes: <any known instability>
```

This is your ground truth. Every subsequent run is compared against this.

---

## Step 3: Controlled Comparison Protocol

When comparing two versions (A vs B), these MUST be identical across both runs:

| Parameter | Must Match |
|---|---|
| Batch file | Same file, same cases, same order |
| `--max-critic-rounds` | Identical — different values = different experiment |
| `--sleep-seconds` | Identical |
| `--retry-cooldown-seconds` | Identical |
| Model | Identical |
| Temperature | Identical |
| Seed | Identical |
| Time of day | Best effort — avoid running during peak API load |

**If any parameter differs, the comparison is not valid for causal claims.**

Document every run with a `_metadata.txt`:
```
commit: <hash> <message>
command: <full command>
batch_file: <path>
max_critic_rounds: <N>
sleep_seconds: <N>
model: <model>
temperature: <N>
seed: <N>
started_at: <ISO timestamp>
results: <per-case outcomes>
pass_rate: N/M
```

---

## Step 4: Interpreting Results

**A change is an improvement if:**
- Pass rate increased AND
- No previously-passing easy case regressed

**A change is neutral if:**
- Pass rate unchanged within ±1 case (could be noise)

**A change is a regression if:**
- Any previously-stable easy case now fails
- Pass rate dropped

**A result is inconclusive if:**
- Only 1-2 runs available (need ≥3 for statistical signal)
- Parameters differed between runs
- Determinism not verified

---

## Step 5: Bisect Protocol (for regressions)

When a regression appears and you don't know which change caused it:

1. **Identify the last known-good commit** (last run where all easy cases passed)
2. **Identify the first known-bad commit** (first run where regression appeared)
3. **List all commits between them that touched LLM behavior** (prompts, node logic, routing)
4. **Binary search:** checkout midpoint commit, run stability batch, check result
5. **Repeat** until you find the first bad commit

**Key insight:** Not all commits affect LLM behavior. Commits that only change display logic, docs, or non-LLM utilities can be skipped in the bisect.

**Document the bisect:**
```
last_good: <commit> — <description> — <pass_rate>
first_bad: <commit> — <description> — <pass_rate>
bisect_result: <commit> — <what it changed> — confirmed by <N> runs
```

---

## Step 6: Statistical Thresholds

For production systems, define minimum acceptable pass rates per case difficulty:

| Difficulty | Minimum pass rate across N runs |
|---|---|
| Easy | ≥ 4/5 runs |
| Medium | ≥ 3/5 runs |
| Hard | ≥ 2/5 runs (aspirational) |

A single run proving nothing is the most common mistake. Budget for multiple runs before claiming improvement.

**Cost estimation before running:**
```
N cases × M LLM calls per case × P runs × cost_per_call = total cost
```

Calculate this before starting any validation cycle. If cost is prohibitive, reduce P (runs) or N (cases) but document the tradeoff.

---

## Known Failure Modes

| Failure | Symptom | Fix |
|---|---|---|
| Different `max-critic-rounds` between runs | Hard case passes with 3 rounds, fails with 1 round — looks like regression | Always match this parameter |
| Entity-specific prompt examples | Parser produces same phrasing as example every run, breaks specific test cases | Remove entity-specific examples from prompts |
| Parallel validation runs | Cascading 429 errors, partial runs, misleading results | Run sequentially only |
| Comparing across API version updates | Model behavior changes independent of your code | Pin model version, note date of any OpenAI updates |