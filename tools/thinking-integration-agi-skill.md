# Thinking, AGI Patterns & Integration Protocols Skills

## Structured Thinking Skill

### Chain-of-Thought Protocol
```
For any complex problem, always:

1. RESTATE — "The task is to..." (ensures correct understanding)
2. DECOMPOSE — "This breaks into: A, B, C..."
3. CONSTRAINTS — "I must not..., I must ensure..."
4. ASSUMPTIONS — "I'm assuming X because..."
5. APPROACH — "I'll solve this by..."
6. EXECUTE — step-by-step, show work
7. VERIFY — "Does the output satisfy all constraints?"
8. COMMUNICATE — clear summary for the human
```

### First Principles Thinking
```
Instead of: "How do others solve X?"
Ask:        "Why does X need solving? What is the minimal solution?"

Example — API latency problem:
Analogy thinking: "Use a cache like Redis"  ← copies what others do
First principles:
  - Why is it slow? (measure first — profile)
  - Is the data actually changing? (maybe it's immutable → cache at CDN)
  - Is the query wrong? (missing index → 1000x faster)
  - Is the architecture wrong? (synchronous where async would work)
```

### The Feynman Technique for Code
```
If you can't explain it simply, you don't understand it yet.

To understand unfamiliar code:
1. Read it through once without stopping
2. Close it and try to write pseudocode from memory
3. Return to code, identify gaps in your mental model
4. Explain each section aloud as if teaching a 10-year-old
5. Find analogies: "this list is like a queue at a bank..."
```

---

## Integration Protocols Skill

### REST vs GraphQL vs gRPC
```
REST:
  ✅ Simple, cacheable, widely understood
  ✅ Great for public APIs and CRUD operations
  ❌ Over-fetching (GET /users returns all fields even if you need 2)
  ❌ Multiple round trips for related data

GraphQL:
  ✅ Client specifies exact fields needed
  ✅ Single endpoint, multiple resources in one request
  ❌ Complex caching, N+1 queries (use DataLoader)
  ❌ Overhead for simple APIs

gRPC:
  ✅ 5-10× faster than REST (binary Protobuf encoding)
  ✅ Streaming (server-push, bidirectional)
  ✅ Strong typing via .proto files
  ❌ Not human-readable, needs tools
  Use for: internal service-to-service, real-time, IoT
```

### Webhook Integration
```python
import hashlib, hmac, json
from fastapi import Request, HTTPException

WEBHOOK_SECRET = os.environ["WEBHOOK_SECRET"]

def verify_webhook(payload: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(secret.encode(), payload, hashlib.sha256).hexdigest()
    # GitHub uses "sha256=<hex>", Stripe uses "t=...,v1=..."
    provided = signature.split("=")[-1]  # strip prefix
    return hmac.compare_digest(expected, provided)

@app.post("/webhook/github")
async def github_webhook(request: Request):
    payload   = await request.body()
    signature = request.headers.get("X-Hub-Signature-256","")
    if not verify_webhook(payload, signature, WEBHOOK_SECRET):
        raise HTTPException(403, "Invalid signature")
    event = json.loads(payload)
    if request.headers.get("X-GitHub-Event") == "push":
        await handle_push(event)
    return {"ok": True}
```

### Message Queue (Redis Streams)
```python
import redis, json, asyncio

r = redis.Redis.from_url(os.environ["REDIS_URL"])

# Producer
def publish(stream: str, event: dict):
    r.xadd(stream, {"data": json.dumps(event)})

# Consumer (with consumer group for load balancing)
def consume(stream: str, group: str, consumer: str):
    try:
        r.xgroup_create(stream, group, id="0", mkstream=True)
    except redis.ResponseError: pass  # group exists

    while True:
        messages = r.xreadgroup(group, consumer, {stream: ">"}, count=10, block=5000)
        for _, msgs in messages or []:
            for msg_id, fields in msgs:
                try:
                    event = json.loads(fields[b"data"])
                    process(event)
                    r.xack(stream, group, msg_id)
                except Exception as e:
                    print(f"Failed {msg_id}: {e}")
                    # Dead letter queue
                    r.xadd(f"{stream}:failed", {"id": msg_id, "error": str(e)})
                    r.xack(stream, group, msg_id)
```

### OAuth2 Integration
```python
import httpx, secrets, hashlib, base64

class OAuth2Client:
    def __init__(self, client_id, client_secret, redirect_uri, auth_url, token_url):
        self.client_id     = client_id
        self.client_secret = client_secret
        self.redirect_uri  = redirect_uri
        self.auth_url      = auth_url
        self.token_url     = token_url

    def get_auth_url(self, scopes: list[str]) -> tuple[str, str]:
        state    = secrets.token_urlsafe(32)
        verifier = secrets.token_urlsafe(64)
        challenge = base64.urlsafe_b64encode(
            hashlib.sha256(verifier.encode()).digest()).rstrip(b"=").decode()
        url = (f"{self.auth_url}?response_type=code"
               f"&client_id={self.client_id}"
               f"&redirect_uri={self.redirect_uri}"
               f"&scope={'+'.join(scopes)}"
               f"&state={state}"
               f"&code_challenge={challenge}&code_challenge_method=S256")
        return url, verifier  # store verifier in session

    def exchange_code(self, code: str, verifier: str) -> dict:
        r = httpx.post(self.token_url, data={
            "grant_type":    "authorization_code",
            "client_id":     self.client_id,
            "client_secret": self.client_secret,
            "redirect_uri":  self.redirect_uri,
            "code":          code,
            "code_verifier": verifier,
        })
        r.raise_for_status()
        return r.json()
```

---

## AGI Patterns — Advanced Agent Architecture

### Recursive Self-Improvement Loop
```python
class SelfImprovingAgent:
    """Agent that evaluates and refines its own outputs."""

    def __init__(self, llm, max_refinements=3):
        self.llm = llm
        self.max_refinements = max_refinements

    def generate(self, task: str) -> str:
        draft = self.llm.complete(f"Task: {task}\nSolution:")
        for i in range(self.max_refinements):
            critique = self.llm.complete(
                f"Task: {task}\n\nSolution:\n{draft}\n\n"
                f"Critique this solution. What is wrong or could be improved? "
                f"Be specific. If it's good enough, say 'APPROVED'.")
            if "APPROVED" in critique.upper():
                break
            draft = self.llm.complete(
                f"Task: {task}\n\nPrevious solution:\n{draft}\n\n"
                f"Critique:\n{critique}\n\nImproved solution:")
        return draft
```

### Multi-Modal Reasoning
```python
def reason_with_image(image_path: str, question: str,
                       tools: list, llm) -> str:
    """Agent that reasons over image + text + tools."""
    import base64
    img_b64 = base64.b64encode(open(image_path,"rb").read()).decode()

    messages = [{
        "role": "user",
        "content": [
            {"type": "image", "source": {"type": "base64",
             "media_type": "image/jpeg", "data": img_b64}},
            {"type": "text", "text":
             f"Analyze the image and answer: {question}\n"
             f"Use available tools if you need additional information."}
        ]
    }]

    # Run agent loop with tools
    while True:
        resp = llm.messages.create(model="claude-sonnet-4-20250514",
                                    max_tokens=2000, tools=tools,
                                    messages=messages)
        if resp.stop_reason == "end_turn":
            return next(b.text for b in resp.content if b.type=="text")
        # Handle tool calls
        results = [{"type":"tool_result","tool_use_id":b.id,
                     "content":dispatch_tool(b.name,b.input)}
                   for b in resp.content if b.type=="tool_use"]
        messages.extend([{"role":"assistant","content":resp.content},
                          {"role":"user","content":results}])
```

---

## README Index — All Skills in This Package
```
skills-mega/
├── core/
│   ├── python-skill.md          Python read/write, async, decorators, testing
│   ├── java-skill.md            Java 17+, streams, Spring Boot, concurrency
│   ├── react-nodejs-skill.md    React hooks, context, router, Express, Vitest
│   └── flet-design-skill.md     Flet Python mobile app, navigation, forms, APK
│
├── frontend/
│   └── frameworks-skill.md      Flutter/Dart, MERN, Vue+Pinia, CSS, TypeScript
│
├── backend/
│   ├── backend-design-skill.md  Architecture reading, REST design, JWT, DI
│   └── database-security-skill.md  SQL injection, encryption, audit, backups
│
├── ai/
│   ├── model-training-skill.md  PyTorch, HuggingFace, LoRA/QLoRA, GGUF, ipynb
│   ├── dataset-generation-skill.md  Synthetic data, augmentation, HF Hub
│   ├── rag-memory-skill.md      RAG pipeline, hybrid search, vector memory
│   └── ocr-vision-3d-skill.md  Tesseract, Claude vision, Three.js, LayoutLM
│
├── systems/
│   ├── assembly-skill.md        x86-64, ARM64, inline C ASM, GDB debugging
│   └── linux-debug-skill.md     Bash, Linux commands, sandbox, Rich CLI, GDB
│
├── architecture/
│   └── strategic-architecture-skill.md  ADR, CQRS, events, micro-frontends
│
├── security/
│   ├── supply-chain-security-skill.md  Dependency pinning, SBOM, CI/CD hardening
│   └── security-research-skill.md     Defensive analysis, OWASP, penetration testing
│
├── automation/
│   ├── scraping-deepsearch-report-skill.md  Scraper, deep research, reports, doc extract
│   └── social-media-automation-skill.md     YouTube/Instagram/Facebook APIs, FFmpeg
│
└── tools/
    └── thinking-integration-agi-skill.md  (this file) Thinking, OAuth2, queues, AGI
```
