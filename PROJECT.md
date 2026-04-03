---
name: project-memory
description: "Core memory skill used by all other skills. Read at the start of every skill activation to understand project state. Write after any decision is confirmed or task is completed. Manages PROJECT.md in the user's repo root — the single source of truth for project decisions, configuration, and progress."
---

# Project Memory

## Overview

`PROJECT.md` is the single source of truth for a project. Every skill reads it at activation to understand the current state. Every skill that makes decisions or completes tasks writes to it after user confirmation.

**Location:** repo root — `PROJECT.md`

**When to read:** At the start of every skill activation. Read only the relevant section, not the entire file.

**When to write:** After the user confirms a decision or completes a task. Never write mid-task or speculatively.

**Rule:** Always tell the user when you've updated `PROJECT.md`. One line is enough: `"Updated PROJECT.md — [what was recorded]."`

---

## PROJECT.md Format

This is the canonical format. Every project using these skills should have a `PROJECT.md` in the repo root with this structure. Sections that haven't been decided yet are left empty or marked `Not decided yet`.

```markdown
# PROJECT.md

## Project Overview
**Name:** [project name]
**Description:** [one sentence — what it does and who it's for]
**Status:** [Planning / In Progress / Deployed]
**Last Updated:** YYYY-MM-DD

---

## System Design
<!-- Written by: ai-system-design -->

**What it does:** [plain language description]
**Who it's for:** [user type]
**Key user flow:** [how a user interacts with it, step by step]
**AI components:** [what AI does in the system]
**Non-AI components:** [what is handled by deterministic logic]

---

## Architecture
<!-- Written by: agent-architecture -->

**Agent framework:** [LangGraph / AutoGen / CrewAI / custom / none]
**Architecture type:** [workflow / single agent with nodes / multi-agent]
**Key components:** [list of nodes/agents and their responsibilities]
**Reasoning:** [why this architecture was chosen]

---

## Models
<!-- Written by: model-selection -->

**Deployment type:** [API / local / hybrid]

| Component | Model | Provider/Tool | Temperature | Notes |
|---|---|---|---|---|
| [component name] | [model] | [provider] | [temp] | [notes] |

**Fine-tuning:** [None / In progress / Completed — details]
**Estimated cost per query:** [amount]

---

## Database
<!-- Written by: database-design -->

**Database type:** [PostgreSQL / MongoDB / SQLite / hybrid]
**Vector store:** [pgvector / Pinecone / Chroma / none]
**Key tables/collections:** [list with one-line description each]
**Migration tool:** [Alembic / Flyway / manual / none]

---

## Memory System
<!-- Written by: memory-system -->

**Memory types used:** [in-context / episodic / semantic / procedural]
**Context window strategy:** [fixed window / token budget / summarization]
**Storage:** [where memories are stored]
**Retrieval strategy:** [recency / similarity / hybrid]

---

## Deployment
<!-- Written by: devops -->

**Target:** [Railway / Render / Fly.io / Docker+VPS / GCP Cloud Run / AWS ECS / Kubernetes]
**Environment:** [production URL or description]
**CI/CD:** [GitHub Actions / none / other]
**Branch strategy:** [main → production / staging → staging / etc.]
**Rollback procedure:** [how to roll back]

---

## Progress Log
<!-- Written by: experiment-driven-development, prompt-change-management, regression-testing, code-review -->

### [YYYY-MM-DD] [Task ID] — [Short Title]
**What changed:** [one sentence]
**Outcome:** [what improved or was confirmed]
**Limitations:** [what is still open]

### [YYYY-MM-DD] [Task ID] — [Short Title]
...

---

## Open Decisions
<!-- Any skill can write here -->
<!-- Remove items when they are decided -->

- [ ] [decision that hasn't been made yet]
- [ ] [decision that hasn't been made yet]

---

## Known Issues
<!-- Any skill can write here -->

| ID | Issue | Status | Discovered |
|---|---|---|---|
| KI-1 | [description] | Open / Resolved | YYYY-MM-DD |

```

---

## Read Protocol

When a skill activates, read only the relevant section of `PROJECT.md`:

| Skill | Read These Sections |
|---|---|
| `ai-system-design` | Project Overview, System Design |
| `agent-architecture` | System Design, Architecture |
| `prompt-design` | Architecture, Models, Progress Log |
| `model-selection` | System Design, Models |
| `database-design` | System Design, Database |
| `memory-system` | Architecture, Database, Memory System |
| `devops` | Architecture, Deployment |
| `experiment-driven-development` | Architecture, Progress Log, Open Decisions, Known Issues |
| `prompt-change-management` | Models, Progress Log |
| `regression-testing` | Models, Progress Log |
| `code-review` | Architecture, Progress Log |
| `agent-integration` | Architecture, Deployment |
| `critic-judge-design` | Architecture, Models |

**If PROJECT.md does not exist:** Tell the user — "I don't see a `PROJECT.md` in this repo yet. Would you like me to create one?" Then create it with the template above, filling in only what is already known from context.

---

## Write Protocol

### After a decision is confirmed

Write to the relevant section. Replace `Not decided yet` with the actual decision. Update `Last Updated` date.

Example — after `model-selection` completes:
```markdown
## Models
<!-- Written by: model-selection -->
<!-- Last updated: 2026-04-03 -->

**Deployment type:** Hybrid (API for main pipeline, local for fallback)

| Component | Model | Provider/Tool | Temperature | Notes |
|---|---|---|---|---|
| Parser | claude-sonnet-4-5 | Anthropic API | 0 | Main decomposition |
| Critic | claude-sonnet-4-5 | Anthropic API | 0 | Sufficiency judgment |
| Fallback | llama3.1:8b | Ollama (local) | 0 | Used when API unavailable |

**Fine-tuning:** None
**Estimated cost per query:** ~$0.02 (2-3 critic rounds, 5-8 LLM calls)
```

### After a task is completed

Append to Progress Log. Never edit old entries — always append.

```markdown
### 2026-04-03 R3.2 — FTS OR Recall Fix
**What changed:** Added OR-join rule to SQL Generator prompt for FTS queries without named entity
**Outcome:** MapleHarvest case improved from 0 rows to 5 rows. Core smoke: 7/9 sufficient.
**Limitations:** Aureum case still has over-constrained FTS; tracked in KI-16
```

### After a code review

Append a summary to Progress Log:
```markdown
### 2026-04-03 Code Review — slack handler + runner
**What changed:** Reviewed interfaces/slack/handler.py and interfaces/runner.py
**Outcome:** 0 CRITICAL, 4 HIGH (all resolved in commit 6a1b913), 14 MEDIUM (11 resolved)
**Limitations:** MAX_SQL_RESULT_ROWS not enforced in executor — deferred
```

---

## Confirming Writes

After every write to `PROJECT.md`, tell the user exactly what was recorded:

```
Updated PROJECT.md:
- Models section: Claude Sonnet as main model, Llama 3.1 8B as local fallback
- Progress Log: R3.2 FTS OR Recall Fix — 7/9 sufficient
```

Keep it short. The user can read `PROJECT.md` for details.

---

## PROJECT.md Initialization

When creating `PROJECT.md` for the first time, fill in what you know and leave the rest as `Not decided yet`:

```bash
# Create from template
touch PROJECT.md
```

Then populate from context — if you're in the middle of `ai-system-design`, fill in Project Overview and System Design. Leave everything else empty. Other skills will fill in their sections as the project progresses.

---

## Keeping PROJECT.md Clean

- **Never duplicate information** — if something is in the database schema file, reference it, don't copy it
- **Progress Log is append-only** — never edit old entries
- **Open Decisions are removed when decided** — not archived, just removed
- **Known Issues are updated in place** — change status from Open to Resolved when fixed
- **Sections written by specific skills** have a `<!-- Written by: skill-name -->` comment — respect this ownership