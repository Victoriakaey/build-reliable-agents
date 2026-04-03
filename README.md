# build-reliable-agents

**An engineering operating system for building reliable LLM agents.**

15 skills for Claude Code covering the full agent development lifecycle. Each skill was extracted from building a production LangGraph agent with a Critic-driven retrieval loop. Framework-agnostic — the principles apply to any agent architecture.

---

## Why This Exists

These are real failure modes that led to the skills in this collection.

**Critic produced confident but wrong judgments.** Input was organized per sub-question. When the answer appeared in a different sub-question's results, the Critic concluded "not found" — the input structure primed per-section reasoning. Fix was restructuring to a flat evidence pool, not changing the prompt. → `critic-judge-design`

**Prompt change improved one case, regressed three others.** No baseline existed. No regression batch ran before committing. The process failure wasn't the prompt edit — it was the lack of a safety net. → `prompt-change-management`, `regression-testing`

**Agent looped because tool observations were too lossy.** Search returned 50 raw rows. The LLM couldn't find the signal, so it retried with rephrased queries — indefinitely. The tool's output format made the correct next action unrecoverable. → `harness-design`

**Verdict-before-reasoning caused post-hoc justification.** The output schema put the boolean verdict before the reasoning field. LLMs generate left-to-right — the model committed to a verdict first, then rationalized. Moving one field fixed it. → `critic-judge-design`

---

## What's Included

Skills activate automatically based on what you're doing.

| Skill | When to use |
|---|---|
| `ai-system-design` | Have an idea but don't know where to start |
| `problem-exploration` | Facing a problem with multiple possible approaches |
| `agent-architecture` | Deciding architecture — workflow vs agent, single vs multi |
| `prompt-design` | Writing a new prompt, or diagnosing one producing wrong output |
| `experiment-driven-development` | Starting any implementation task |
| `prompt-change-management` | About to change a prompt |
| `regression-testing` | Comparing two system versions |
| `critic-judge-design` | Designing a Judge, Critic, or Evaluator component |
| `harness-design` | Agent misbehaves — wrong tools, missed data, loops |
| `agent-integration` | Connecting agent to a web app, API, or third-party platform |
| `database-design` | Designing a schema (general or AI-specific) |
| `model-selection` | Choosing a model, API vs local, fine-tuning decisions |
| `memory-system` | Designing how an agent remembers across sessions |
| `devops` | Deploying to production, CI/CD, monitoring |
| `code-review` | Reviewing code with structured severity levels |

All skills read from and write to `PROJECT.md` — a shared state contract at the repo root that keeps decisions, progress, and known issues in one place across sessions.

---

## Philosophy

**Treat every change as an experiment, not a fix.** Every change needs before/after measurement. Regressions are expected.

**Structure beats instructions.** Input organization and output schema shape LLM reasoning more than prompt rules. Fix structure before adding rules.

**Evidence before code.** Start with real system output. Write down the failure mode and hypothesis before touching any file.

**One job per component.** If it does two things, split it.

---

## Installation

### Claude Code

```bash
/plugin marketplace add Victoriakaey/build-reliable-agents
/plugin install build-reliable-agents@build-reliable-agents
```

### claude.ai (Manual)

```bash
git clone https://github.com/Victoriakaey/build-reliable-agents.git
cp -r build-reliable-agents/skills/ /mnt/skills/user/build-reliable-agents/
```

### Update

```bash
/plugin update build-reliable-agents
```

---

## Contributing

Contributions welcome. Each skill should be grounded in real observed failures, not theoretical best practices. Follow the existing SKILL.md structure.

---

## License

MIT License. See [LICENSE](LICENSE) for details.
