# Strategic Decisions, Cross-Domain Debugging & Architecture Skills

## Strategic Decision Framework

### Technology Selection Matrix
```
Score each option 1–5 on:
- Team expertise      (don't learn on the job for critical systems)
- Community & docs    (will you be stuck when problems arise?)
- Performance fit     (does it match the load profile?)
- Operational cost    (hosting, licensing, maintenance)
- Longevity           (still supported in 5 years?)
- Migration cost      (how hard to change later?)

→ Highest weighted score wins
→ If scores are close: pick the boring, proven option
```

### When NOT to Build Custom
```
Build custom when:
  ✅ Core competitive differentiator
  ✅ Existing tools don't fit after serious evaluation
  ✅ Team has deep expertise in the domain

Use off-the-shelf when:
  ✅ Authentication/Auth (use Auth0, Keycloak, Supabase)
  ✅ Search (use Elasticsearch, Typesense, Meilisearch)
  ✅ Email delivery (use SendGrid, Postmark, SES)
  ✅ Payments (use Stripe — never roll your own)
  ✅ File storage (use S3/R2 — never local disk in prod)
```

### Architecture Decision Record (ADR)
```markdown
# ADR-001: Use PostgreSQL as Primary Database

## Status: Accepted

## Context
We need a primary database for the platform supporting 100K users.
Candidates: PostgreSQL, MySQL, MongoDB.

## Decision
PostgreSQL with proper indexes and connection pooling (PgBouncer).

## Rationale
- ACID compliance needed for financial data
- JSONB support covers semi-structured data without MongoDB complexity
- Team has 4 years PostgreSQL experience
- Excellent cloud support (RDS, Supabase, Neon)
- Full-text search via pg_trgm avoids Elasticsearch for now

## Consequences
+ Strong consistency guarantees
+ Single database to operate
- Horizontal scaling requires Citus or read replicas if >10M rows

## Alternatives Rejected
- MongoDB: flexible schema not needed, ACID concerns
- MySQL: less feature-rich, weaker JSON support
```

---

## Cross-Domain Debugging

### Debugging Methodology
```
1. REPRODUCE — get a minimal, consistent reproduction
2. ISOLATE    — is it frontend/backend/DB/network/infra?
3. BISECT     — binary search: which commit/change broke it?
4. HYPOTHESIZE — form ONE theory before changing code
5. VERIFY     — prove the fix without breaking anything else
6. DOCUMENT   — write why the bug existed, not just what changed
```

### Network Layer Debug
```bash
# Is the service even reachable?
curl -v http://localhost:8080/health          # verbose HTTP
curl -I https://api.example.com              # just headers
telnet api.example.com 443                   # TCP connect test
nslookup api.example.com                     # DNS resolution
traceroute api.example.com                   # hop-by-hop routing

# Check what's listening
ss -tulpn | grep :8080
lsof -i :8080

# Capture traffic
tcpdump -i lo -A 'port 8080'                 # loopback HTTP
wireshark -k -i eth0 -f "port 443"          # HTTPS (needs decryption key)
```

### Database Debug
```sql
-- Find slow queries (PostgreSQL)
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC LIMIT 20;

-- Active connections + locks
SELECT pid, state, wait_event_type, wait_event, query
FROM pg_stat_activity WHERE state != 'idle';

-- Lock waits
SELECT * FROM pg_locks WHERE NOT granted;

-- Table bloat
SELECT relname, n_dead_tup, n_live_tup,
       round(n_dead_tup::numeric/nullif(n_live_tup+n_dead_tup,0)*100, 2) AS dead_pct
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;

-- Explain with analyze
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = 42 ORDER BY created_at DESC LIMIT 10;
```

### Frontend + Backend Debug
```javascript
// Network tab alternative — log all fetch calls
const origFetch = window.fetch;
window.fetch = (...args) => {
    console.log("FETCH:", args[0]);
    return origFetch(...args).then(r => {
        console.log("RESPONSE:", r.status, args[0]);
        return r;
    });
};

// React component re-render audit
import { useRef } from "react";
function useRenderCount(name) {
    const count = useRef(0);
    count.current++;
    console.log(`${name} rendered ${count.current} times`);
}
```

---

## Architectural Innovation Patterns

### Event-Driven Architecture
```python
from dataclasses import dataclass
from typing import Callable, Any
import asyncio, json
from collections import defaultdict

@dataclass
class Event:
    type:    str
    payload: dict
    version: str = "1.0"

class EventBus:
    def __init__(self):
        self._handlers: dict[str, list[Callable]] = defaultdict(list)
        self._middleware: list[Callable] = []

    def on(self, event_type: str):
        def decorator(fn: Callable):
            self._handlers[event_type].append(fn)
            return fn
        return decorator

    async def emit(self, event: Event):
        # Run middleware
        for mw in self._middleware:
            event = await mw(event) or event
        # Run handlers
        handlers = self._handlers.get(event.type, [])
        await asyncio.gather(*[h(event) for h in handlers])

bus = EventBus()

@bus.on("user.registered")
async def send_welcome_email(event: Event):
    print(f"Sending welcome to {event.payload['email']}")

@bus.on("user.registered")
async def create_default_settings(event: Event):
    print(f"Creating settings for user {event.payload['id']}")
```

### CQRS (Command Query Responsibility Segregation)
```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

# Commands — change state
@dataclass
class CreateOrderCommand:
    user_id: int; product_ids: list[int]; shipping_address: str

# Queries — read state (separate read models)
@dataclass
class GetUserOrdersQuery:
    user_id: int; page: int = 1; limit: int = 20

class CommandHandler(ABC):
    @abstractmethod
    def handle(self, command) -> dict: ...

class QueryHandler(ABC):
    @abstractmethod
    def handle(self, query) -> dict: ...

class CreateOrderHandler(CommandHandler):
    def handle(self, cmd: CreateOrderCommand) -> dict:
        # Write to main DB
        order = OrderRepository.create(cmd.user_id, cmd.product_ids)
        # Publish event for read model update
        bus.emit(Event("order.created", {"id": order.id, "user_id": cmd.user_id}))
        return {"order_id": order.id}
```

### Micro-Frontend Architecture
```javascript
// Shell app — loads microfrontends
const MODULES = {
    dashboard: "https://dashboard.myapp.com/remoteEntry.js",
    settings:  "https://settings.myapp.com/remoteEntry.js",
};

async function loadModule(name) {
    const container = await import(MODULES[name]);
    await container.init(sharedScope);
    const factory = await container.get("./App");
    return factory();
}

// Module Federation (webpack.config.js)
module.exports = {
    plugins: [new ModuleFederationPlugin({
        name: "dashboard",
        filename: "remoteEntry.js",
        exposes: { "./App": "./src/App" },
        shared: { react: { singleton: true }, "react-dom": { singleton: true } },
    })],
};
```

### Hexagonal Architecture
```
         Adapters (in)          Core           Adapters (out)
┌─────────────────────┐  ┌──────────────┐  ┌───────────────────┐
│ HTTP REST Controller│──│              │──│ PostgreSQL Repo    │
│ GraphQL Resolver    │──│  Domain      │──│ Redis Cache        │
│ gRPC Service        │──│  Services    │──│ SendGrid Email     │
│ CLI Command         │──│  Entities    │──│ S3 Storage         │
│ Message Consumer    │──│  Use Cases   │──│ Stripe Payments    │
└─────────────────────┘  └──────────────┘  └───────────────────┘
         ↑                                           ↑
  Ports (interfaces)                        Ports (interfaces)

Rule: Domain code imports NOTHING from adapters.
      Adapters implement domain-defined interfaces (ports).
```

## Unwritten Context Skills

### Reading Between the Lines
```
User says: "Can you make this faster?"
→ May mean: "It takes 10 seconds and users are complaining"
→ Ask: What's the current time? What's acceptable? Which operation?

User says: "This doesn't work"
→ Ask: What should it do? What does it do instead? Error message?

User says: "We need to scale this"
→ Ask: Current load? Expected load? Bottleneck already identified?
→ Don't assume: "scale" could mean 10x or 1000x — very different solutions

Code comments "// TODO: fix this properly"
→ The current code is a hack — understand WHY before changing
→ The author knew it was wrong — find the constraints that forced the hack
```

### Code Archaeology
```bash
# Who wrote this and why?
git log --follow -p src/payment.py         # full history with diffs
git blame -L 42,56 src/payment.py          # who wrote each line
git log --grep="payment" --all             # commits mentioning payment
git log --author="Ali" --since="2024-01-01"

# What changed around a bug introduction?
git bisect start
git bisect bad HEAD              # current is broken
git bisect good v1.2.0           # this version was fine
# Git automatically checks out midpoint commits for you to test
```
