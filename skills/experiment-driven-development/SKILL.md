---
name: experiment-driven-development
description: "Use at the start of any implementation task on an LLM system. A 10-step process from evidence collection to git commit. Prevents the most common failure mode in LLM development: writing code based on assumptions instead of observed system behavior. Use for any change — prompt tuning, node logic, routing, architecture — not just experiments."
---

# Experiment-Driven Development

## The Core Problem

LLM systems fail in non-obvious ways. A change that fixes one case often breaks another. Without a structured process, development becomes: change something → run it → it looks better → commit → discover regression three sessions later.

**The discipline:** Never write code based on assumptions. Every change starts with evidence from real system output, has a written hypothesis, and ends with a validation run that checks positive controls.

---

## Quick Reference Checklist

Use this before starting and before committing.

### Before Starting
- [ ] Task ID and exit criteria confirmed
- [ ] Baseline test count recorded (`pytest`)
- [ ] Positive controls identified (cases that must not regress)
- [ ] Real log evidence collected (not assumptions)
- [ ] Failure mode, evidence, suspected layer, hypothesis written down
- [ ] Anti-overfitting check passed
- [ ] Decision gate passed (evidence is sufficient to proceed)
- [ ] User confirmed the plan

### Before Committing
- [ ] `py_compile` passed on changed files
- [ ] `pytest` passed (count ≥ baseline)
- [ ] Focused validation batch ran and saved to logs
- [ ] User confirmed result interpretation
- [ ] Positive controls did not regress
- [ ] Iteration journal entry written (with Limitations section)
- [ ] Known issues updated (if applicable)
- [ ] Experiment tracker updated (if applicable)
- [ ] Exit criteria verified against roadmap

---

## Step 0 — Pre-Flight

Before touching any code or documentation:

1. **Confirm task identity**
   - What is the task ID and scope?
   - What are the exit criteria?
   - Which known issues or future improvements does this task address?

2. **Record baseline test count**
   ```bash
   python -m pytest tests/ -v
   ```
   Write down the count (e.g., "47 passed"). This is your floor — it must not decrease.

3. **Identify positive controls**
   - Which cases currently pass and must continue to pass after this change?
   - Write them down explicitly before proceeding.
   - These are your regression canaries.

---

## Step 1 — Evidence First

**Do not plan or propose changes based on assumptions. Start with real system output.**

1. **Check existing logs** for the relevant failure cases.
   - Read the actual output — not just summaries, but the full node outputs.
   - What exactly does the system produce now?

2. **If no relevant logs exist, run a baseline batch.**
   - Save to `logs/<task-id>-baseline/<timestamp>/`
   - This is your "before" snapshot.

3. **Extract the concrete failure mode.**
   - What exactly does the system produce?
   - What should it produce instead?
   - Be specific: quote log lines, identify exact fields.

4. **Hunt for counterexamples.**
   - Is there a case in the same failure family that works correctly?
   - If yes: what is different between the working case and the failing case?
   - This contrast reveals root cause more precisely than the failure alone.

5. **Distinguish transient vs real failures.**
   - API rate limits, connection errors, quota exhaustion → invalid evidence, re-run after resolved
   - Do not draw conclusions from transient failures

---

## Step 2 — Failure-First Write-Up

Write this down before proposing any change:

```
Observation:    [What was seen in logs — raw evidence, quoted log lines]
Evidence:       [Paths to specific log files]
Suspected Layer:[Which component is responsible: parser / generator / 
                 validator / executor / critic / router / synthesizer]
Hypothesis:     [Precise, testable: "If we change X, then Y should 
                 improve because Z"]
Anti-Overfitting Check:
  - Does this fix target a reusable failure pattern, or one specific case?
  - Would this change make sense without having seen the specific test case?
  - Does the fix apply to other cases with the same structural properties?
```

**The anti-overfitting check is not optional.** A change that only fixes one specific case by memorizing its expected answer is not a real fix.

---

## Step 3 — Decision Gate

After the write-up, explicitly decide: is the evidence sufficient to proceed?

**Sufficient means:**
- At least one clear failing case with identified root cause
- At least one counterexample or positive control for comparison
- Suspected layer clearly identified, not guessed

**If insufficient:** run more cases, return to Step 1. Do not proceed to coding with weak evidence. A deferred experiment is better than a wrong one.

**If sufficient:** proceed to Step 4 or Step 5.

---

## Step 4 — Brainstorm (If Design Decisions Needed)

Skip if the task is a straightforward fix with no design ambiguity. Note that it was skipped.

If design decisions are needed:

1. **Create a brainstorm log:** `docs/working-notes/<topic>-brainstorm-log.md`

2. **Ask clarifying questions one at a time.**
   - Must reach 95% confidence before writing code
   - One question per message — wait for answer before asking next

3. **Record each Q&A immediately** using this format:
   ```
   Question asked: [exact text]
   User's verbatim response: [exact words, never paraphrase]
   Interpreted decision: [implementation-level detail]
   ```
   Write each entry into the brainstorm log as part of the same response.
   Never batch updates to the end of the session.

4. **For each option considered, document:**
   - What it does
   - Trade-offs table (files changed, cost, risk, effectiveness, determinism)
   - Why rejected or chosen
   - Unresolved gaps (things that must be figured out before implementation)

---

## Step 5 — Plan and Confirmation

Write the plan before writing any code:

```
Proposed Change:    [What will change, in which files]
Why This Change:    [Connect back to hypothesis from Step 2]
Positive Controls:  [Named explicitly — which cases must not regress]
Focused Batch:      [Which test input file for validation]
Scope Boundary:     [What this task will NOT change]
```

**Wait for explicit user confirmation before writing any code.**
Do not interpret silence or partial acknowledgment as confirmation.

---

## Step 6 — Code

1. **Smallest safe patch.**
   - Change one narrow layer or one tightly related cluster.
   - Do not combine unrelated concerns in one commit:
     - Node boundary refactors
     - Prompt tuning
     - Routing changes
     - Output format changes
   - Unless the interaction between them is explicitly the point of the task.

2. **Write deterministic tests for the changed behavior.**
   - Happy path for the new behavior
   - Edge cases
   - Fallback / no-op behavior when the new logic does not apply

---

## Step 7 — Validation Ladder

Execute in this exact order. Do not skip steps.

### 7.1: Syntax Check
```bash
python -m py_compile <changed-files>
```

### 7.2: Deterministic Tests
```bash
python -m pytest tests/ -v
```
Count must be ≥ Step 0 baseline.

### 7.3: Focused Runtime Batch
```bash
python <entry-point> \
  --batch-file test-inputs/<task-file>.txt \
  --logs-dir logs/<task-id>-<slug>/<timestamp>/
```
Save output. This is your "after" snapshot.

### 7.4: Present Results — Before Writing Documentation

Show the user:

**a) Before/After Comparison** — what changed and what stayed the same

**b) Key Output Excerpts** — the most relevant sections from logs (not full dumps)

**c) Positive Control Status** — for each identified positive control:
- "Case X: still passing ✅" or "Case X: REGRESSED ❌"

**d) Ambiguous Cases** — if any case's behavior changed in a way that is unclear:
- "This case changed from X to Y. My interpretation is Z. Do you agree?"

**e) Proposed Limitations** — what was NOT fixed, what new issues surfaced

**Wait for user confirmation before proceeding to documentation.**

### 7.5: Broader Regression Batch (Only If 7.4 Confirmed)
Run the full test suite. Save to a separate log directory. Step 8 (documentation) waits for 7.4 confirmation but does NOT require 7.5 to complete — 7.5 can run in parallel with documentation if time-constrained.

### When Validation Fails

| Situation | Action |
|---|---|
| Positive control regressed | Blocker. Diagnose and fix, or revert and record finding. Do not force through. |
| Target case did not improve | Re-examine hypothesis. Was the suspected layer correct? |
| Unexpected behavior in unrelated case | Flag to user. Determine if caused by this change or pre-existing. |
| Transient API failure | Run is invalid. Re-run after resolved. Do not use as evidence. |

---

## Step 8 — Documentation

Only after user confirms validation results.

### Iteration Journal Entry (required for every task)

```markdown
## Iteration N — [Short Title]

**Date:** YYYY-MM-DD

### Observations
[Raw evidence from real test runs. Quote log lines. What was seen, not interpreted.]

### Motivation
[Why was this change proposed? Connect observations to the decision.
Include user feedback verbatim if it triggered the change.]

### What Changed
[Which files, what logic, what decision points and how they were resolved.]

### Impact
**Positive:** [What concretely improved]
**Limitations:** [What new problems surfaced, or what is still not solved.
Limitations are not optional — record honestly.]

### Close-Out
- **Current Capability:** [What the system can now do after this task]
- **Issue Status:** [Which issues were fixed, narrowed, or intentionally not touched]
- **Runtime Output Summary:** [What the pipeline produced on focused validation]
- **Validation Artifacts:** [Links to saved log files]
```

### Other Documentation (update only when applicable)

| Document | Update When |
|---|---|
| Prompt change log | Any prompt was modified |
| Known issues | Task materially changed an issue's status |
| Experiment tracker | Task involved critic / loop / routing / stopping behavior |
| Design clarifications | Node boundaries or output contracts changed |
| Future improvements | A related entry's status changed |
| Case-family coverage matrix | A case family's behavior changed |

---

## Step 9 — Closure Check

Before committing:

1. Re-read the task's exit criteria. Is it actually met?
2. If exit criteria are NOT met — record as Residual Risk in the iteration journal. Do not mark the task as closed.

---

## Step 10 — Git

```bash
git add <specific-files>          # Never git add .
git commit -m "<prefix>: <message>"  # prefixes: experiment: / fix: / tests: / docs:
git push origin <current-branch>
```

One task = one commit. Record the commit ID in the experiment tracker if applicable.

---

## The Discipline in One Sentence

**Evidence → Hypothesis → Smallest change → Validation → Honest documentation.**

Every shortcut in this process creates a debt that compounds. A skipped decision gate means a wrong fix. A skipped positive control check means a silent regression. A skipped Limitations section means the next session starts blind.