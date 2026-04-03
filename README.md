# build-reliable-agents

Building LLM agents is easy. Building ones that work consistently is not.

This is a collection of engineering skills for Claude that brings structure to every stage of AI application development — from the first vague idea to production deployment. The skills were extracted from building a production LangGraph agent with a Critic-driven retrieval loop. The patterns reflect what was learned the hard way.

Framework-agnostic. Examples use LangGraph but the principles apply to AutoGen, CrewAI, or any custom agent architecture.

---

## How it works

The skills activate automatically based on context — no special commands needed.

Have a rough idea but don't know where to start? `ai-system-design` guides you through a conversation to figure out what you're actually building, where AI fits in, and how the full system should be designed — in plain language, then as a structured spec Claude can build from.

About to change a prompt? `prompt-change-management` makes you document what you're changing and why, identifies the blast radius, and runs a validation batch before committing.

Getting inconsistent results across runs? `regression-testing` gives you a controlled comparison protocol so you can tell the difference between "this change made things better" and "I got lucky on this run."

Building a Critic or Judge component? `critic-judge-design` covers the failure modes nobody talks about — how input organization primes the wrong reasoning pattern, why verdict-before-reasoning produces post-hoc justification, and when to split one overloaded component into two.

Ready to deploy? `devops` guides you through choosing a deployment target, containerizing, setting up CI/CD, and defining a rollback strategy.

There's more to it — 14 sub-skills covering the full lifecycle.

---

## What's Included

| Sub-Skill | Activates When |
|---|---|
| `ai-system-design` | Have an idea but don't know where to start or how to design the system |
| `problem-exploration` | Facing a problem with multiple possible solutions, or a previous approach failed |
| `agent-architecture` | Deciding architecture — workflow vs agent, single vs multi, tool-use vs specialized components |
| `prompt-design` | Writing a prompt for any LLM component, or diagnosing one producing wrong outputs |
| `experiment-driven-development` | Starting any implementation task — 10-step process from evidence to git commit |
| `prompt-change-management` | About to change a prompt — safe iteration with regression detection |
| `regression-testing` | Comparing two system versions, or debugging non-deterministic behavior |
| `critic-judge-design` | Designing any LLM-as-Judge, Critic, or Evaluator component |
| `agent-integration` | Connecting an agent to a web app, REST API, WebSocket, Webhook, or message queue |
| `database-design` | Designing a schema — general or AI-specific (conversations, embeddings, agent state) |
| `model-selection` | Choosing a model, deciding API vs local, or evaluating whether to fine-tune |
| `memory-system` | Designing how an agent remembers and retrieves information across sessions |
| `devops` | Deploying to production, setting up CI/CD, monitoring, rollback strategy |
| `code-review` | Claude reviews your code and produces a report, or guides you through reviewing |

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
cp -r build-reliable-agents/llm-systems/ /mnt/skills/user/llm-systems/
```

### Verify Installation

Start a new session and ask something that should trigger a skill — for example, *"I have an idea for an AI product but don't know where to start"* or *"my Critic keeps producing inconsistent results."* The relevant skill should activate automatically.

---

## Philosophy

**LLM behavior is non-deterministic. Treat every change as an experiment, not a fix.**
Every change needs a before/after measurement. Regressions are expected. Prompt changes have blast radius beyond the component they touch.

**Structure beats instructions.**
Input organization, output schema design, and system/user prompt split shape LLM reasoning more than any rule you write. Fix structure before adding rules.

**Evidence before code.**
Never write code based on assumptions. Start with real system output. Write down the failure mode, the suspected layer, and a testable hypothesis before touching any file.

**One job per component.**
If a component does two things, split it. The extra LLM call is cheaper than the debugging cost of an overloaded prompt.

---

## Contributing

If you use these skills and find gaps or new patterns worth capturing, contributions are welcome. The goal is to keep each skill grounded in real observed failures — not theoretical best practices.

New sub-skills should follow the existing SKILL.md structure: frontmatter `description` field, clear trigger conditions, and concrete examples drawn from real system behavior.

1. Fork the repository
2. Create a branch for your skill
3. Follow the `prompt-design` skill's structure as a reference
4. Submit a PR

---

## Community

- **Issues**: https://github.com/Victoriakaey/build-reliable-agents/issues
- **Email**: jd.victoria.work@gmail.com

---

## License

MIT License. See [LICENSE](LICENSE) for details.