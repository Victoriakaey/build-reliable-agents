---
name: build-reliable-agents
description: "A collection of engineering skills for building reliable LLM-based systems and full-stack AI applications. Use when starting a new AI project, designing an agent pipeline, writing or diagnosing prompts, exploring solutions to a technical problem, making prompt changes, debugging non-deterministic behavior, designing a database, selecting a model, deploying to production, reviewing code, or connecting an agent to a full-stack application."
---

# build-reliable-agents

## Overview

This skill collection captures patterns battle-tested in production LLM system development. Each sub-skill is a self-contained process for one recurring problem. They can be used independently or in combination.

## When to Use Which Sub-Skill

| Situation | Sub-Skill |
|---|---|
| Have an idea but don't know where to start | `ai-system-design` |
| Facing a problem with multiple possible solutions | `problem-exploration` |
| Previous approach failed, need to reconsider | `problem-exploration` |
| Deciding agent architecture (single vs multi, tool-use vs nodes) | `agent-architecture` |
| Writing a new prompt from scratch | `prompt-design` (Mode 1) |
| Existing prompt producing unstable or wrong outputs | `prompt-design` (Mode 2) |
| Starting any implementation task | `experiment-driven-development` |
| About to change a prompt | `prompt-change-management` |
| Need to compare two system versions | `regression-testing` |
| Designing a Critic, Judge, or Evaluator component | `critic-judge-design` |
| Connecting agent to a web app, API, or third-party platform | `agent-integration` |
| Designing or improving agent tools, observations, and evaluation signals | `harness-design` |
| Agent picks wrong tool, ignores data, or loops without progress | `harness-design` (diagnosis ladder) |
| Designing a database schema | `database-design` |
| Choosing a model or deciding API vs local | `model-selection` |
| Designing how an agent remembers information | `memory-system` |
| Deploying to production or setting up CI/CD | `devops` |
| Reviewing code — Claude reviews or guided self-review | `code-review` |
| Starting a new LLM project from scratch | `ai-system-design` → `agent-architecture` → `model-selection` → `database-design` → `experiment-driven-development` |
| System behaving inconsistently across runs | `regression-testing` → `prompt-design` (Mode 2) |
| Critic/Judge producing unstable judgments | `critic-judge-design` → `prompt-design` (Mode 2) |
| Agent works locally but fails when integrated | `agent-integration` → `devops` |
| Agent misbehaves — wrong tools, missed data, silent failures | `harness-design` → `experiment-driven-development` |

## Core Principle Across All Sub-Skills

**LLM behavior is non-deterministic. Treat every change as an experiment, not a fix.**

This means:
- Every change has a before/after measurement
- Regressions are expected and must be detected early
- Prompt changes have blast radius beyond the component they touch
- Deterministic components (validators, routers, formatters) are always preferable to LLM components when both can do the job

## Sub-Skills

- [`ai-system-design/SKILL.md`](ai-system-design/SKILL.md) — Go from a vague idea to a full system design, guided by conversation
- [`problem-exploration/SKILL.md`](problem-exploration/SKILL.md) — Exhaust the solution space before committing to any approach
- [`agent-architecture/SKILL.md`](agent-architecture/SKILL.md) — Agent architecture decision framework
- [`prompt-design/SKILL.md`](prompt-design/SKILL.md) — Write or diagnose prompts for any LLM component
- [`experiment-driven-development/SKILL.md`](experiment-driven-development/SKILL.md) — 10-step process from evidence to git commit
- [`prompt-change-management/SKILL.md`](prompt-change-management/SKILL.md) — Safe prompt iteration with regression detection
- [`regression-testing/SKILL.md`](regression-testing/SKILL.md) — Controlled comparison protocol for LLM systems
- [`critic-judge-design/SKILL.md`](critic-judge-design/SKILL.md) — Designing reliable LLM-as-Judge components
- [`agent-integration/SKILL.md`](agent-integration/SKILL.md) — Connect agents to full-stack apps, APIs, and third-party platforms
- [`database-design/SKILL.md`](database-design/SKILL.md) — Schema design for general and AI-specific systems
- [`model-selection/SKILL.md`](model-selection/SKILL.md) — API vs local, model selection, fine-tuning decisions
- [`memory-system/SKILL.md`](memory-system/SKILL.md) — Design how agents remember and retrieve information
- [`devops/SKILL.md`](devops/SKILL.md) — Deploy to production, CI/CD, monitoring, rollback
- [`harness-design/SKILL.md`](harness-design/SKILL.md) — Design action space, tool boundaries, observation format, and evaluation signals
- [`code-review/SKILL.md`](code-review/SKILL.md) — Claude reviews your code or guides you through reviewing