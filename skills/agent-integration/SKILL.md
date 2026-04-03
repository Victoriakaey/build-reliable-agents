---
name: agent-integration
description: "Use when connecting an LLM agent to a full-stack application, external API, or third-party platform. Covers four integration patterns (REST, WebSocket/SSE, Webhook, Message Queue), interface design, reliability, security, and observability. Framework-agnostic — guides you to the right pattern for your situation, then gives concrete implementation direction for your chosen stack."
---

# Agent Integration

## The Core Problem

LLM agents are slow (10-60 seconds per run), stateful, and non-deterministic. Standard web API patterns assume fast, stateless, deterministic handlers. Naively connecting the two produces: blocked event loops, silent failures, duplicate runs, leaked state across users, and no visibility into what went wrong.

**The discipline:** Choose the right integration pattern for your latency and delivery requirements before writing any code. Most integration bugs come from using the wrong pattern, not from implementing the right one badly.

---

## Step 1: Choose Your Integration Pattern

Answer these two questions:

**Q1: Who initiates the request?**
- Your frontend / user → REST or WebSocket/SSE
- A third-party platform (Slack, GitHub, Stripe, etc.) → Webhook
- A background job or queue → Message Queue

**Q2: How long does the agent take, and does the user need live progress?**
- < 3 seconds, no progress needed → REST (synchronous)
- 5-60 seconds, user needs progress → WebSocket or SSE
- Any duration, fire-and-forget → Message Queue
- Third-party with strict ack timeout → Webhook + async background

| Pattern | Best For | Avoid When |
|---|---|---|
| REST (sync) | Fast agents (<3s), simple request/response | Agent takes >3s — will timeout or block |
| REST (async + polling) | Medium agents (3-30s), client can poll | Real-time progress needed |
| SSE | Long agents, one-way progress stream to browser | Bidirectional communication needed |
| WebSocket | Long agents, bidirectional (user can cancel/redirect) | Simple one-way progress is enough |
| Webhook | Third-party platform pushes events to your server | You initiate the request |
| Message Queue | Background jobs, high volume, retryable work | Low latency required |

---

## Step 2: Interface Design

### Pattern A: REST (Synchronous)

Only use if agent completes in < 3 seconds. Otherwise use async + polling.

```
POST /agent/run
Body: { query: string, session_id: string }
Response: { answer: string, citations: [...], run_id: string }
```

**Stack recommendations:**
- Python: FastAPI with `async def` handler
- Node: Express or Hono with async handler
- Do NOT run the agent synchronously in the request handler if it takes > 1s — use a thread pool

```python
# FastAPI — run blocking agent in thread pool
@app.post("/agent/run")
async def run_agent(request: AgentRequest):
    result = await asyncio.to_thread(run_pipeline, request.query)
    return result
```

---

### Pattern B: REST (Async + Polling)

For agents that take 5-30 seconds. Client submits job, polls for status.

```
POST /agent/run          → { job_id: string }          (immediate)
GET  /agent/run/{job_id} → { status: queued|running|done|failed, result?: ... }
```

**Implementation:**
- On POST: create job record (in-memory dict, Redis, or DB), start background task, return job_id immediately
- Background task: run agent, update job record on completion/failure
- On GET: return current job record

```python
jobs = {}  # or Redis

@app.post("/agent/run")
async def submit(request: AgentRequest):
    job_id = str(uuid.uuid4())
    jobs[job_id] = {"status": "queued"}
    asyncio.create_task(run_and_store(job_id, request.query))
    return {"job_id": job_id}

@app.get("/agent/run/{job_id}")
async def poll(job_id: str):
    return jobs.get(job_id, {"status": "not_found"})
```

**Polling interval:** Tell the client to poll every 2-3 seconds. Exponential backoff after 30s.

---

### Pattern C: Server-Sent Events (SSE)

For agents where you want to stream progress to a browser. One-way: server → client.

```
GET /agent/stream?query=...
Content-Type: text/event-stream

data: {"phase": "parsing", "message": "Analyzing your question..."}
data: {"phase": "querying", "message": "Querying database (2 sub-questions)..."}
data: {"phase": "evaluating", "message": "Evaluating evidence (round 1)..."}
data: {"phase": "done", "answer": "...", "citations": [...]}
```

**Implementation:**

```python
# FastAPI SSE
from fastapi.responses import StreamingResponse

@app.get("/agent/stream")
async def stream(query: str):
    async def event_generator():
        async for event in run_pipeline_streaming(query):
            yield f"data: {json.dumps(event)}\n\n"
    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

**Frontend (browser):**
```javascript
const source = new EventSource(`/agent/stream?query=${encodeURIComponent(query)}`);
source.onmessage = (e) => {
    const event = JSON.parse(e.data);
    if (event.phase === "done") source.close();
    updateUI(event);
};
```

**Reconnection:** SSE reconnects automatically on drop. Use `Last-Event-ID` header if you need resume support.

---

### Pattern D: WebSocket

For agents where the user might need to send mid-run input (cancel, clarify, redirect).

```
WS /agent/ws

Client → Server: { type: "run", query: "...", session_id: "..." }
Server → Client: { type: "progress", phase: "parsing", message: "..." }
Server → Client: { type: "progress", phase: "querying", message: "..." }
Server → Client: { type: "done", answer: "...", citations: [...] }
Server → Client: { type: "error", message: "..." }

Client → Server: { type: "cancel" }  ← user can send mid-run
```

**Implementation:**

```python
# FastAPI WebSocket
@app.websocket("/agent/ws")
async def websocket_endpoint(ws: WebSocket):
    await ws.accept()
    try:
        while True:
            msg = await ws.receive_json()
            if msg["type"] == "run":
                async for event in run_pipeline_streaming(msg["query"]):
                    await ws.send_json(event)
            elif msg["type"] == "cancel":
                break  # handle cancellation
    except WebSocketDisconnect:
        pass
```

**Use SSE instead if** you don't need bidirectional communication — SSE is simpler, works over HTTP/2, and doesn't require a persistent connection manager.

---

### Pattern E: Webhook

For third-party platforms that push events to your server (Slack, GitHub, Stripe, Linear, etc.).

**The 3-second problem:** Most platforms require an HTTP 200 within 3 seconds. Your agent takes 20 seconds. Never run the agent synchronously in the webhook handler.

```
Platform → POST /webhook/slack     (must return 200 within 3s)
         ← 200 OK                  (ack immediately)
         → background task starts  (run agent async)
         → POST back to platform   (send result when done)
```

**Implementation:**

```python
@app.post("/webhook/slack")
async def slack_webhook(request: Request):
    # 1. Verify signature FIRST — reject invalid requests immediately
    await verify_signature(request)

    # 2. Parse payload
    payload = await request.json()

    # 3. Ack immediately — do NOT await the agent
    asyncio.create_task(handle_slack_event(payload))
    return {"ok": True}  # 200 returned before agent starts

async def handle_slack_event(payload):
    result = await asyncio.to_thread(run_pipeline, payload["query"])
    await post_to_slack(payload["channel"], result)
```

**Signature verification** (required for security — see Step 4):
- Slack: HMAC-SHA256 of request body with signing secret, check `X-Slack-Signature`
- GitHub: HMAC-SHA256 with webhook secret, check `X-Hub-Signature-256`
- Stripe: timestamp + payload signature, check `Stripe-Signature`
- Generic: implement the same pattern for any platform

**Deduplication:** Platforms retry on failure. Use the event ID to deduplicate:
```python
processed_events = set()  # or Redis SET with TTL

async def handle_slack_event(payload):
    event_id = payload.get("event_id")
    if event_id in processed_events:
        return  # already handled
    processed_events.add(event_id)
    ...
```

---

### Pattern F: Message Queue

For background jobs, high volume, or when you need retryable work with guaranteed delivery.

```
API Server  →  Queue (Redis/RabbitMQ/SQS)  →  Worker  →  Result Store
                                                       →  Notify client (webhook/WS)
```

**When to use:**
- Agent runs are expensive and should not be lost on server restart
- High volume — more jobs than your API server can handle
- You need retry logic with backoff on failure
- Jobs can be processed out of order

**Stack recommendations:**
- Python: Celery + Redis, or ARQ (async), or AWS SQS + Lambda
- Node: BullMQ + Redis, or AWS SQS
- Simple: Redis LIST with LPUSH/BRPOP (works for low volume)

```python
# Simple Redis queue with ARQ
async def run_agent_job(ctx, query: str, job_id: str):
    result = await run_pipeline(query)
    await store_result(job_id, result)

# Enqueue from API
@app.post("/agent/run")
async def submit(request: AgentRequest):
    job_id = str(uuid.uuid4())
    await redis.enqueue_job("run_agent_job", request.query, job_id)
    return {"job_id": job_id}
```

---

## Step 3: Reliability

### Async discipline

**Rule:** Never run a blocking LLM call on the async event loop thread.

```python
# WRONG — blocks the event loop, no other requests can be handled
@app.post("/run")
async def run(request):
    result = run_pipeline(request.query)  # blocking call
    return result

# RIGHT — offload to thread pool
@app.post("/run")
async def run(request):
    result = await asyncio.to_thread(run_pipeline, request.query)
    return result
```

### Timeout handling

Set explicit timeouts at every layer:

```python
# HTTP client timeout
async with httpx.AsyncClient(timeout=30.0) as client:
    response = await client.post(...)

# Agent pipeline timeout
try:
    result = await asyncio.wait_for(
        asyncio.to_thread(run_pipeline, query),
        timeout=60.0
    )
except asyncio.TimeoutError:
    return {"error": "Agent timed out", "partial": get_partial_result()}
```

### Error handling — fail loudly, not silently

```python
# WRONG — swallowed exception, downstream gets empty result, no log
try:
    result = run_node(state)
except Exception:
    return {}

# RIGHT — log, propagate meaningfully
try:
    result = run_node(state)
except Exception as e:
    logger.error("Node failed: %s", e, exc_info=True)
    raise  # or return explicit error state
```

**One error path, not two:** After an error, do not continue to the success path.

```python
# WRONG — on_error runs, then on_complete runs, user gets two messages
try:
    result = run_pipeline(query)
    renderer.on_complete(result)
except Exception as e:
    renderer.on_error(e)
    renderer.on_complete(result)  # still runs! result is empty

# RIGHT
try:
    result = run_pipeline(query)
    renderer.on_complete(result)
except Exception as e:
    renderer.on_error(e)
    return  # stop here
```

### State isolation (multi-user)

Every user session must have isolated state. Never use module-level mutable state for per-session data.

```python
# WRONG — shared across all users
session_state = {}  # module-level

# RIGHT — keyed by session/user ID
@app.post("/run")
async def run(request: AgentRequest):
    session_id = request.session_id  # or from auth token
    state = get_or_create_session(session_id)
    result = run_pipeline(request.query, state)
    ...
```

For LangGraph: use a checkpointer keyed by `session_id` or `thread_id`.

### Idempotency

Protect against duplicate requests (network retries, user double-clicks, platform retries):

```python
# Store completed job results by request fingerprint
completed = {}  # or Redis with TTL

@app.post("/run")
async def run(request: AgentRequest):
    fingerprint = hash((request.query, request.session_id))
    if fingerprint in completed:
        return completed[fingerprint]  # return cached result

    result = await asyncio.to_thread(run_pipeline, request.query)
    completed[fingerprint] = result
    return result
```

---

## Step 4: Security

### Authentication

| Pattern | Recommended Auth |
|---|---|
| REST API (your frontend) | JWT or session token in Authorization header |
| REST API (other services) | API key in header, validated at startup |
| Webhook (third-party) | Platform-specific signature verification (HMAC) |
| WebSocket | Auth token in connection handshake, not in messages |
| Message Queue | IAM roles (cloud) or connection credentials |

**Never** authenticate via query parameter — it ends up in server logs.

### Signature verification for webhooks

Always verify before processing. Reject before parsing body.

```python
import hmac, hashlib

def verify_github_signature(body: bytes, signature: str, secret: str) -> bool:
    expected = "sha256=" + hmac.new(
        secret.encode(), body, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)

@app.post("/webhook/github")
async def github_webhook(request: Request):
    body = await request.body()
    sig = request.headers.get("X-Hub-Signature-256", "")
    if not verify_github_signature(body, sig, GITHUB_WEBHOOK_SECRET):
        raise HTTPException(status_code=401)
    # now safe to process
```

### Secret management

- Never hardcode secrets in source
- Use environment variables loaded at startup
- Validate presence at startup — raise immediately if missing, do not silently default to empty string

```python
# WRONG — silently defaults to "", fails at runtime with cryptic error
SLACK_SECRET = os.getenv("SLACK_SIGNING_SECRET", "")

# RIGHT — fails fast at startup with clear message
SLACK_SECRET = os.environ["SLACK_SIGNING_SECRET"]  # raises KeyError if missing
```

### Input validation

Validate and sanitize all external input before passing to agent:

```python
# Normalize: strip HTML entities, trim whitespace, remove control characters
import html

def normalize_input(text: str) -> str:
    text = html.unescape(text)       # decode &amp; &#39; etc.
    text = text.strip()
    text = " ".join(text.split())    # normalize whitespace
    return text
```

For SQL-backed agents: use read-only DB connections and keyword blocklists as defense in depth (see `agent-architecture` skill).

---

## Step 5: Observability

### What to log at the integration layer

Every request should produce a structured log entry:

```python
logger.info("agent_run", extra={
    "run_id": run_id,
    "session_id": session_id,
    "query_length": len(query),
    "pattern": "webhook",           # which integration pattern
    "source": "slack",              # which platform
    "duration_ms": duration,
    "outcome": "success",           # success / error / timeout
    "llm_calls": call_count,
    "critic_rounds": round_count,
})
```

### Trace continuity across agent + API boundary

The agent's internal trace (LangSmith, etc.) and the API layer's logs should share a `run_id` so you can correlate:

```python
run_id = str(uuid.uuid4())
# Pass to agent
result = run_pipeline(query, run_id=run_id)
# Include in API response and logs
return {"run_id": run_id, "answer": result.answer}
```

### Health endpoint

Every integration service should expose a health endpoint:

```python
@app.get("/health")
async def health():
    return {
        "status": "ok",
        "version": VERSION,
        "db_connected": check_db(),
        "llm_reachable": check_llm(),
    }
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Blocking call in async handler | All requests freeze when agent is running | `asyncio.to_thread()` |
| No ack before agent starts (webhook) | Platform retries, agent runs twice | Return 200 before starting agent |
| No signature verification (webhook) | Anyone can POST to your endpoint | Verify HMAC before processing |
| Silent secret default to "" | Cryptic auth failure at runtime | Validate at startup, raise if missing |
| Module-level mutable state | Users see each other's data | Key all state by session_id |
| No idempotency | Duplicate runs from retries | Fingerprint + cache completed results |
| Swallowed exceptions | Silent failures, user gets empty response | Log + propagate, single error path |
| On_error + on_complete both run | User gets two messages after error | Return early after on_error |
| No timeout | Hung agent holds connection open indefinitely | `asyncio.wait_for()` with explicit timeout |