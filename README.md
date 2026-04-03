# build-reliable-agents

**An engineering operating system for building reliable LLM agents.**

Building agents is easy. Building ones that work consistently is not. The hard part isn't making a demo — it's preventing regressions when you change a prompt, catching silent failures before users do, and knowing *why* your agent broke instead of guessing.

This is a set of 15 engineering skills for Claude Code that bring structure to every stage of agent development — from first idea to production deployment. Each skill was extracted from building a production LangGraph agent with a Critic-driven retrieval loop. The patterns reflect what was learned the hard way.

Framework-agnostic. Examples use LangGraph but the principles apply to any agent framework.

---

## Why This Exists

These are real failure modes from production agent development. Each one led to a skill in this collection.

### Critic produced confident but wrong judgments

The Critic node evaluated evidence per sub-question. When the answer appeared in a *different* sub-question's results, it concluded "not found" — because the input structure primed per-section reasoning.

```
Before (organized by sub-question):
  SQ1: "Find patch window"     → 0 rows         → Critic: "not found"
  SQ2: "Find rollback artifacts" → 3 rows (contains patch window!)  → Critic: ignores

After (flat evidence pool):
  Original question: ...
  All retrieved evidence: [rows from all sub-questions merged]
  → Critic: "found — row 7 contains patch window"
```

Fix wasn't a prompt change. It was restructuring the input. → `critic-judge-design`

### Prompt change improved one case, regressed three others

Changed the SQL generator prompt to handle a new query pattern. Manual testing looked good.

```
Day 1:  New prompt → target query now returns 5 rows  ✓
Day 3:  User reports "Verdant Bay" query broken       ✗
        "MapleHarvest" query returns 0 rows            ✗
        "Aureum" query times out                       ✗
```

**Root cause:** Prompt changes have blast radius. A rule added for one query pattern implicitly changed how the LLM weighted existing rules for other patterns. The regression was invisible because there was no baseline — no set of known-good cases to run before committing. The process failure wasn't the prompt edit itself; it was committing without a regression batch. → `prompt-change-management`, `regression-testing`

### Agent looped because tool observations were too lossy

The agent called a search tool, got back raw data, and couldn't find the signal it needed. It re-invoked the same tool with slight variations — indefinitely.

```
Step 1 → search("project timeline") → 50 rows, no summary
Step 2 → agent can't find answer in noise
Step 3 → search("project timeline details") → 48 rows, mostly same data
Step 4 → search("timeline project") → same results
→ loop until token limit

Fix: observation returns top-5 rows + "12 more rows matched" + schema summary
→ agent finds answer in step 1
```

**Root cause:** The agent's observation layer returned raw database rows with no structure or prioritization. The LLM couldn't distinguish signal from noise in 50 undifferentiated rows, so it did the only thing it could: retry with a rephrased query, hoping for better results. The loop wasn't a prompt problem or a planning problem — it was a **harness design problem**. The tool's output format made the correct next action unrecoverable from the observation. → `harness-design`

### Verdict-before-reasoning caused post-hoc justification

The Critic's output schema put the boolean verdict field before the reasoning field.

```
Before (verdict first):
  { "sufficient": true, "reasoning": "The evidence shows..." }
  → LLM commits to verdict, then generates reasoning to justify it

After (reasoning first):
  { "reasoning": "SQ1 has no direct match, but SQ2 row 3...", "sufficient": true }
  → LLM reasons through evidence, then derives verdict
```

**Root cause:** LLMs generate tokens left-to-right. When the schema forces the verdict token to be emitted before the reasoning tokens, the model commits to a position and then rationalizes it — the same way a human who states a conclusion first will unconsciously filter evidence to support it. The output schema field ordering *is* the reasoning order. No prompt instruction ("think carefully before judging") can override the autoregressive generation constraint. → `critic-judge-design`

---

## How It Fits Together

```
                        ┌─────────────────────────────────┐
                        │        Your Agent System         │
                        │                                  │
                        │   User ──→ Agent ──→ Tools       │
                        │              │                   │
                        │          Critic/Judge             │
                        │              │                   │
                        │         Evaluation Loop           │
                        └──────────────┬──────────────────┘
                                       │
              ┌────────────────────────┼────────────────────────┐
              │                        │                        │
     ┌────────▼────────┐    ┌─────────▼─────────┐    ┌────────▼────────┐
     │  Design Skills   │    │  Iteration Skills  │    │  Infra Skills   │
     │                  │    │                    │    │                 │
     │ ai-system-design │    │ experiment-driven  │    │ agent-integr.   │
     │ agent-architect. │    │ prompt-change-mgmt │    │ database-design │
     │ prompt-design    │    │ regression-testing │    │ model-selection │
     │ critic-judge     │    │ harness-design     │    │ memory-system   │
     │ problem-explor.  │    │ code-review        │    │ devops          │
     └────────┬─────────┘    └─────────┬──────────┘    └────────┬────────┘
              │                        │                        │
              └────────────────────────┼────────────────────────┘
                                       │
                            ┌──────────▼──────────┐
                            │    PROJECT.md        │
                            │  (state contract)    │
                            │                      │
                            │  decisions · models  │
                            │  progress · issues   │
                            └─────────────────────┘
```

Skills are organized in three layers. **Design skills** help you make architectural decisions. **Iteration skills** enforce disciplined change-test-commit cycles. **Infra skills** handle the integration and deployment surface. All skills read from and write to `PROJECT.md` — the shared state contract that keeps them coherent.

---

## What's Included

Each skill is a self-contained process. They activate automatically based on what you're doing — no special commands needed.

| Sub-Skill | You bring | It enforces | You get |
|---|---|---|---|
| `ai-system-design` | A vague product idea | Guided conversation to figure out what AI actually does in the system | Structured spec Claude can build from |
| `problem-exploration` | A technical problem with multiple possible approaches | Exhaustive solution space mapping before committing | Ranked options with tradeoffs |
| `agent-architecture` | System requirements | Decision framework: workflow vs agent, single vs multi, tool-use vs nodes | Architecture decision with reasoning |
| `prompt-design` | A new prompt to write, or an existing one producing wrong output | Input/output structure design before writing instructions | Prompt with structural integrity |
| `experiment-driven-development` | Any implementation task | 10-step process: evidence → hypothesis → code → validation → commit | Change with before/after measurement |
| `prompt-change-management` | A prompt you're about to modify | Blast radius analysis, positive control identification, regression batch | Safe prompt iteration |
| `regression-testing` | Two system versions to compare | Controlled comparison protocol with statistical awareness | Evidence of improvement vs luck |
| `critic-judge-design` | A Judge/Critic/Evaluator component to build | Input structure design, output schema ordering, failure mode prevention | Reliable evaluation node |
| `harness-design` | An agent that misbehaves — wrong tools, missed data, loops | 5-layer diagnosis: action space → tool boundaries → observations → eval signals → iteration | Structural fix, not prompt patch |
| `agent-integration` | An agent to connect to a web app, API, or third-party platform | Integration pattern selection, error boundary design | Production-ready connection |
| `database-design` | Schema requirements (general or AI-specific) | Normalized design with AI-specific patterns (conversations, embeddings, state) | Schema with migration strategy |
| `model-selection` | A decision: which model, API vs local, fine-tune or not | Cost/capability/latency tradeoff framework | Model configuration per component |
| `memory-system` | An agent that needs to remember across sessions | Memory type selection, retrieval strategy, storage design | Memory architecture |
| `devops` | Code ready for production | Deployment target selection, containerization, CI/CD, rollback strategy | Deployment pipeline |
| `code-review` | Code to review | Structured review with severity levels | Review report with prioritized fixes |

---

## PROJECT.md — Cross-Skill State Contract

Every agent project accumulates decisions: which architecture, which models, what failed, what's still open. These decisions live in people's heads or scattered across docs until they're forgotten. Then a skill makes a decision that contradicts one made three weeks ago.

`PROJECT.md` is a **state contract with write discipline**. It's not documentation — it's the persistence layer that keeps 15 independent skills coherent across sessions, days, and collaborators.

**Write rules are strict by design:**
- Each section has an owner skill (marked by `<!-- Written by: skill-name -->`)
- No skill writes mid-task — only after user confirmation
- Progress Log is append-only — history is never rewritten
- Open Decisions are removed when decided, not archived

```
System Design ← written by ai-system-design
Architecture  ← written by agent-architecture
Models        ← written by model-selection
Database      ← written by database-design
Memory        ← written by memory-system
Deployment    ← written by devops
Progress Log  ← append-only, written by any skill that completes a task
Open Decisions ← removed when decided
Known Issues   ← updated in place
```

The result: a single source of truth that grows with the project. Every skill reads only what it needs, writes only what it owns, and never makes decisions in a vacuum.

---

## Philosophy

**Treat every change as an experiment, not a fix.**
LLM behavior is non-deterministic. Every change needs a before/after measurement. Regressions are expected. Prompt changes have blast radius beyond the component they touch.

**Structure beats instructions.**
Input organization, output schema design, and system/user prompt split shape LLM reasoning more than any rule you write. Fix structure before adding rules.

**Evidence before code.**
Never write code based on assumptions. Start with real system output. Write down the failure mode, the suspected layer, and a testable hypothesis before touching any file.

**One job per component.**
If a component does two things, split it. The extra LLM call is cheaper than the debugging cost of an overloaded prompt.

---

## Installation

### Claude Code (Plugin Marketplace)

```bash
/plugin marketplace add Victoriakaey/build-reliable-agents
/plugin install build-reliable-agents@build-reliable-agents
```

### Update

```bash
/plugin update build-reliable-agents
```

### claude.ai (Manual)

```bash
git clone https://github.com/Victoriakaey/build-reliable-agents.git
cp -r build-reliable-agents/skills/ /mnt/skills/user/build-reliable-agents/
```

### Verify

Start a new session and say something like *"I have an idea for an AI product but don't know where to start"* or *"my Critic keeps producing inconsistent results."* The relevant skill activates automatically.

---

## Contributing

If you use these skills and find gaps or new patterns worth capturing, contributions are welcome. The goal is to keep each skill grounded in real observed failures — not theoretical best practices.

New sub-skills should follow the existing SKILL.md structure: frontmatter `description` field, clear trigger conditions, and concrete examples drawn from real system behavior.

1. Fork the repository
2. Create a branch for your skill
3. Follow `prompt-design` skill's structure as a reference
4. Submit a PR

---

## Community

- **Issues**: https://github.com/Victoriakaey/build-reliable-agents/issues
- **Email**: jd.victoria.work@gmail.com

---

## License

MIT License. See [LICENSE](LICENSE) for details.
