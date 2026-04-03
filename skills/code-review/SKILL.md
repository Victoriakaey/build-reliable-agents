---
name: code-review
description: "Use when reviewing code — either Claude reviews your code and produces a structured report, or Claude guides you through reviewing someone else's code. Default mode: Claude performs the review and produces a report organized by severity. Second mode: guided self-review with a structured checklist and probing questions. Covers correctness, security, performance, maintainability, and AI-specific concerns for LLM applications."
---

# Code Review

## Two Modes

**Mode 1 (Default): Claude Reviews Your Code**
Provide the code and Claude produces a structured review report organized by severity.
→ Say "review this code" or paste code directly.

**Mode 2: Guided Self-Review**
Claude guides you through reviewing code yourself with a structured checklist and probing questions.
→ Say "help me review this" or "walk me through reviewing this PR."

---

# Mode 1: Claude Reviews Your Code

## How to Provide Code for Review

For best results, provide:
1. The code to review
2. What it's supposed to do (one sentence is enough)
3. Any specific concerns you have (optional)
4. The language/framework if not obvious

## Review Report Format

Claude produces a report in this structure:

```
## Code Review: [filename or description]

### Summary
[2-3 sentence overview of the code quality and main findings]

### CRITICAL Issues
[Issues that will cause bugs, security vulnerabilities, or data loss]

### HIGH Issues
[Issues that will likely cause problems in production]

### MEDIUM Issues
[Issues that reduce quality, maintainability, or performance]

### LOW Issues / Style
[Minor improvements, style suggestions, nitpicks]

### Positive Observations
[What's done well — always include this]

### Recommended Next Steps
[Prioritized list of what to fix first]
```

## What Claude Checks

### Correctness
- Logic errors and off-by-one errors
- Edge cases not handled (empty input, null values, boundary conditions)
- Incorrect assumptions about data types or formats
- Race conditions in concurrent code
- Error handling — are exceptions caught and handled appropriately?
- Return values — are all return paths handled?

### Security
- Hardcoded secrets, API keys, passwords
- SQL injection (string concatenation in queries)
- Input not validated or sanitized before use
- Sensitive data logged or exposed in error messages
- Authentication/authorization checks missing
- Dependencies with known vulnerabilities (flagged if obvious)
- Data sent to external APIs without considering compliance requirements (GDPR, HIPAA, data residency)

### Performance
- N+1 query patterns (loop containing a database query)
- Missing indexes on frequently queried columns
- Unnecessary data loaded (SELECT * when specific columns needed)
- Blocking calls in async context
- Large objects held in memory unnecessarily
- Missing caching where it would clearly help

### Maintainability
- Functions doing more than one thing
- Magic numbers and strings without named constants
- Deeply nested code (> 3 levels)
- Duplicated logic that should be extracted
- Missing or misleading comments on complex logic
- Variable names that don't communicate intent
- Dead code

### AI-Specific (for LLM applications)
- Prompt injection vulnerabilities (user input directly in prompts without sanitization)
- Missing token limit enforcement
- LLM errors not caught (network errors, rate limits, context length errors)
- Sensitive data sent to external API without awareness
- Non-deterministic behavior not accounted for (same input can produce different output)
- Missing fallback when LLM call fails
- Swallowed exceptions in LLM call wrappers
- Blocking LLM calls in async handlers
- No timeout on LLM API calls
- Cost not considered (unbounded loops with LLM calls)

---

# Mode 2: Guided Self-Review

Use this when you want to review code yourself — your own code before committing, a colleague's PR, or code you're inheriting.

## Phase 1: Understand Before Judging

Before looking for problems, understand what the code does.

**Questions to answer first:**
- What is this code supposed to do? (read the PR description, ticket, or ask)
- What are the inputs and outputs?
- What are the external dependencies (database, APIs, other services)?
- What could go wrong at runtime?

If you can't answer these, ask before reviewing.

## Phase 2: Read the Code in the Right Order

Don't read top to bottom — read in order of importance:

1. **Public interface first** — function signatures, class methods, API endpoints. Do they make sense?
2. **Error handling paths** — what happens when things go wrong?
3. **Main logic** — the happy path
4. **Helper functions** — only if the main logic calls them and you need to understand them

## Phase 3: Checklist

Work through this checklist. For each item, either confirm it's fine or flag it.

### Correctness
- [ ] Does the code do what it's supposed to do?
- [ ] Are edge cases handled? (empty input, null/undefined, zero, negative numbers, very large values)
- [ ] Are all error paths handled? (what happens if a dependency fails?)
- [ ] Are there any obvious logic errors?
- [ ] Are async operations awaited correctly?

### Security
- [ ] Are there any hardcoded secrets or credentials?
- [ ] Is user input validated before use?
- [ ] Is user input sanitized before going into SQL queries or prompts?
- [ ] Are sensitive values (passwords, tokens) avoided in logs?
- [ ] Are authentication/authorization checks in place where needed?
- [ ] Is data sent to external APIs (especially LLM providers) compliant with data residency/privacy requirements?

### Performance
- [ ] Are there database queries inside loops?
- [ ] Is more data loaded than needed?
- [ ] Are there blocking operations in async code?
- [ ] Is there anything that will get significantly slower as data grows?

### Maintainability
- [ ] Can you understand what each function does from its name alone?
- [ ] Are there any functions longer than ~50 lines? (flag for potential splitting)
- [ ] Is there duplicated logic that could be extracted?
- [ ] Are complex sections commented?
- [ ] Are magic numbers or strings given names?

### Tests
- [ ] Are there tests for the main logic?
- [ ] Are edge cases tested?
- [ ] Do the tests actually test behavior, or just implementation details?
- [ ] If no tests, is the code at least testable (dependency injection, no hidden globals)?

### AI-Specific (if applicable)
- [ ] Is user input sanitized before being added to a prompt?
- [ ] Are LLM API calls wrapped in try/except?
- [ ] Is there a timeout on LLM calls?
- [ ] Is there a fallback if the LLM call fails?
- [ ] Are token limits enforced?
- [ ] Is there a guard against unbounded LLM call loops?

## Phase 4: Writing the Review

### Tone principles
- **Be specific** — "this function is hard to read" is not actionable. "This function does three things (parse, validate, save) — consider splitting" is.
- **Explain the why** — don't just say what's wrong, say why it matters.
- **Separate must-fix from nice-to-have** — be explicit about severity.
- **Acknowledge what's good** — a review with only negatives is demoralizing and misses important signal.
- **Ask questions instead of making accusations** — "did you consider X?" is better than "you forgot X."

### Comment format

```
[CRITICAL] This SQL query is vulnerable to injection — user input is concatenated
directly into the query string. Use parameterized queries instead:
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))

[HIGH] LLM call on line 47 has no exception handling. If the API is unavailable
or returns a rate limit error, this will crash the entire request. Wrap in
try/except and return a graceful fallback.

[MEDIUM] The process_items() function does both filtering and transformation.
Consider splitting into filter_items() and transform_items() for testability
and clarity.

[LOW] Magic number 50 on line 23 — consider defining MAX_RESULTS = 50 at the
top of the file.

[POSITIVE] Good use of TypedDict for the agent state — makes the state contract
explicit and easy to understand.
```

---

## Severity Definitions

Use these consistently so the person receiving the review knows what must be fixed vs. what's optional.

| Severity | Definition | Must fix before merge? |
|---|---|---|
| **CRITICAL** | Will cause data loss, security vulnerability, or crash in production | Yes — block merge |
| **HIGH** | Will likely cause bugs or failures in production | Yes — fix before merge |
| **MEDIUM** | Reduces quality, maintainability, or performance | Recommended — discuss if not fixing |
| **LOW** | Style, minor improvements, personal preference | Optional |
| **POSITIVE** | Something done well — worth calling out | N/A |

---

## AI Code Review: Extra Checklist

For code that involves LLM calls, agents, or AI pipelines, run this additional checklist:

### Prompt safety
- [ ] User input is never directly concatenated into a prompt without sanitization
- [ ] System prompts are not exposed to users
- [ ] Prompt injection attack surface is minimized

### Reliability
- [ ] Every LLM call has a try/except
- [ ] Every LLM call has a timeout
- [ ] There is a fallback response when the LLM fails
- [ ] Retry logic exists for transient failures (rate limits, network errors)
- [ ] There is a hard limit on retry attempts

### Cost and performance
- [ ] LLM calls are not made inside unbounded loops
- [ ] Token counts are tracked and logged
- [ ] Context window limits are enforced before making calls
- [ ] Expensive LLM calls are not on hot paths that run on every request

### State and determinism
- [ ] LLM outputs are not assumed to be deterministic
- [ ] Structured output parsing has a fallback for malformed responses
- [ ] Agent state is not shared across user sessions
- [ ] Sensitive user data is not persisted in agent state unnecessarily

### Observability
- [ ] LLM calls are logged with input/output (or at least metadata)
- [ ] Errors from LLM calls produce useful log messages
- [ ] Token usage is tracked for cost monitoring