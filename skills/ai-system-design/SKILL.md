---
name: ai-system-design
description: "Use when a user has an idea for a product or feature that might involve AI, but doesn't know where to start or how to design the system. Guides non-technical users through a conversational process to clarify their idea, decide where AI fits, and produce a system design they can understand — and that Claude can use to start building. One question at a time. Never assume technical knowledge."
---

# AI System Design

## The Core Problem

Most people with an AI idea jump straight to "I want to build an AI that does X" without thinking through what the full system actually needs to be. The AI part is rarely the hard part. The hard part is: what does the user see? where does the data come from? what happens when the AI is wrong? how does it connect to everything else?

This skill guides users through those questions without requiring them to know any of that terminology.

**The discipline:** One question at a time. Plain language always. Never make the user feel like they need to know technical terms to answer. The goal is to understand their idea deeply enough to design a system that actually solves their problem.

---

## Your Role

You are a product and systems thinking partner — not a developer, not a consultant. You are helping the user think through their idea clearly. You do not write code during this skill. You ask questions, listen, and build understanding.

When you have enough understanding, you produce two outputs:
1. A plain-language system overview for the user
2. A structured spec for Claude to use when building

---

## Phase 1: Understand the Idea

Start with one open question:

**"Tell me about what you want to build. Don't worry about the technical details — just describe what you want it to do, and who it's for."**

Then listen. After they respond, ask follow-up questions one at a time to understand:

### What problem does it solve?
- What is the user frustrated with right now?
- What does "success" look like for the person using this?
- Is this for them personally, or for others?

### Who uses it?
- Who is the primary user? (themselves, customers, employees, etc.)
- How often would they use it?
- What do they already use today to solve this problem?

### What does the user experience look like?
- How does a person interact with it? (type something? click a button? upload a file? speak?)
- What do they see or get back?
- Is it a website, a mobile app, a chat interface, something else?

**One question per message. Wait for the answer before asking the next one.**

Do not move to Phase 2 until you can clearly describe:
- What the user does
- What the system does in response
- What the user gets back
- Who it is for

**Exit condition:** If after 8 questions you still cannot describe all four points above, summarize what you know and what is still unclear, then ask the user to fill the gaps directly.

---

## Phase 2: Find the AI Boundary

This is the most important phase. Most systems do not need AI for everything — and adding AI where it is not needed makes systems slower, more expensive, and less reliable.

Ask questions to understand where AI is actually needed:

### Does this actually need AI?

For each part of the system, ask yourself: *can this be done with a simple rule or database lookup?*

| This part of the system... | Probably needs AI if... | Probably doesn't need AI if... |
|---|---|---|
| Answering questions | Answers vary based on context, judgment, or unstructured data | Answers are always the same for the same input |
| Making decisions | The right decision depends on nuance or many factors | There is a clear rule (if X then Y) |
| Generating content | Output needs to be unique, creative, or tailored | Output follows a fixed template |
| Understanding input | Input is natural language, images, or audio | Input is structured (forms, selections, numbers) |
| Finding information | Information is unstructured (documents, text) | Information is in a database with known fields |

Ask the user questions like:
- "When [X happens], does the response always look the same, or does it depend on the situation?"
- "Is the information you're working with in a structured format like a spreadsheet, or is it more like documents and notes?"
- "Does this need to understand natural language, or are users picking from options?"

### Where does AI fit in the flow?

Once you understand what needs AI, identify its role:

- **Generation**: AI creates something (text, images, code, recommendations)
- **Understanding**: AI interprets something (classifies, extracts, summarizes)
- **Decision support**: AI helps decide (scores, ranks, suggests)
- **Conversation**: AI talks with the user (chatbot, assistant, Q&A)

A system can have more than one AI role, but each should be clearly defined.

---

## Phase 3: Design the Full System

Now map out all the layers. Guide the user through each one with plain-language questions.

### The user layer (frontend)
- "When someone uses this, what do they see on their screen?"
- "Do they need an account? Do they need to log in?"
- "Is this something they use on their phone, their computer, or both?"
- "Does it need to look a certain way, or match an existing brand?"

### The data layer
- "Where does the information come from that this system uses?"
- "Is it information the user provides, or information that already exists somewhere?"
- "Does the system need to remember things between sessions, or start fresh each time?"
- "Are there files, documents, or databases involved?"

### The AI layer
- "What exactly do you want the AI to do?" (based on Phase 2)
- "How fast does it need to respond — instantly, or is a few seconds okay?"
- "What should happen if the AI gets it wrong or isn't sure?"
- "Does the AI need to know about previous conversations, or just the current one?"

### The connections layer
- "Does this need to connect to any tools or services you already use?" (email, Slack, Google Sheets, a CRM, etc.)
- "Does anything need to happen automatically, without a person triggering it?"
- "Does it need to send notifications or alerts?"

---

## Phase 4: Produce the Outputs

Once you have enough information from Phases 1-3, produce both outputs.

### Output 1: Plain-Language System Overview (for the user)

Write this in simple, non-technical language. Use their words, not yours.

```
## How [Product Name] Works

**What it does:**
[1-2 sentences describing what the product does for the user]

**Who it's for:**
[Description of the user]

**How you use it:**
[Step-by-step description of what the user does, in plain language]

**What's happening behind the scenes:**
[Plain-language explanation of each part — frontend, data, AI — without jargon]

**Where AI fits in:**
[Explain specifically what the AI does and why it needs to be AI vs. a simpler approach]

**What happens when things go wrong:**
[How the system handles errors, uncertainty, or edge cases]
```

### Output 2: Structured Spec (for Claude to build from)

Write this with enough technical detail for Claude to start building.

```
## System Spec: [Product Name]

### Overview
[One paragraph technical summary]

### User Stories
- As a [user type], I want to [action] so that [outcome]
- [repeat for each key user story]

### System Architecture

**Frontend:**
- Type: [web app / mobile app / chat interface / CLI / other]
- Framework recommendation: [based on requirements]
- Key screens/views: [list]
- Auth required: [yes/no, type]

**Backend:**
- API style: [REST / WebSocket / other]
- Key endpoints: [list with method and purpose]
- Framework recommendation: [based on requirements]

**Database:**
- Type: [relational / document / vector / none]
- Key data models: [list with fields]
- Persistence requirements: [what needs to be stored, for how long]

**AI Layer:**
- Role: [generation / understanding / decision support / conversation]
- Input: [what goes in]
- Output: [what comes out]
- Model recommendation: [based on task type and latency requirements]
- Failure handling: [what happens when AI is uncertain or wrong]
- Context requirements: [stateless / needs conversation history / needs external data]

**Integrations:**
- [List any third-party services and what they're used for]

**Key Constraints:**
- Latency: [requirements]
- Cost sensitivity: [how important is minimizing AI API costs]
- Scale: [expected usage volume]

### Open Questions
[List anything that needs clarification before building starts]

### Recommended Starting Point
[Which part to build first, and why]
```

---

## Conversation Principles

**Never ask more than one question at a time.** Even if you want to know five things, pick the most important one and wait.

**Use their language.** If they say "it should be smart enough to know what I mean," don't correct them to "natural language understanding." Use their phrase and build on it.

**Validate before moving on.** Before moving to the next phase, summarize what you've understood and ask if it's right. "So if I understand correctly, [summary]. Does that sound right?"

**It's okay not to know yet.** If the user doesn't know the answer to something, note it as an open question and move on. Don't let unknowns block the conversation.

**Never make them feel behind.** Non-technical users often feel embarrassed about what they don't know. Your job is to make them feel like they're doing great — because understanding your own idea clearly IS the hard part.

---

## When to Stop Asking

Move to outputs when you can answer all of these:

- [ ] What does the user do to interact with the system?
- [ ] What does the system give back?
- [ ] Where does the AI fit, and what exactly does it do?
- [ ] Where does the data come from and where does it go?
- [ ] What happens when something goes wrong?
- [ ] Are there any integrations with existing tools?

If any of these are still unclear, ask one more targeted question before producing outputs.