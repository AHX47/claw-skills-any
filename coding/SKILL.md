# Coding Skill — Claude Best Practices

## Overview
This skill guides Claude to produce production-grade code across all languages and frameworks. Use when the user asks to write, debug, refactor, review, or explain code.

---

## Core Principles

### 1. Understand Before Writing
- Clarify requirements, language, framework, and target environment first.
- Ask about constraints: Python version, browser support, mobile/desktop, performance needs.
- Identify edge cases before writing a single line.

### 2. Code Quality Standards
Always produce code that is:
- **Correct** — handles edge cases, errors, and nulls
- **Readable** — clear names, small functions, comments where non-obvious
- **Maintainable** — DRY, modular, follows language conventions
- **Tested** — include unit tests or at least test snippets
- **Secure** — no hardcoded secrets, validate inputs, no SQL injection risks

### 3. Language-Specific Rules

#### Python
- Use type hints for all function signatures
- Prefer f-strings over `.format()` or `%`
- Use `pathlib` over `os.path`
- Use `dataclasses` or `pydantic` for data models
- Use `contextlib` for resource management
- Async: always `await` coroutines, use `asyncio.gather()` for concurrency
- Error handling: specific exceptions, never bare `except:`

```python
# Good
async def fetch_data(url: str, timeout: int = 10) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(url, timeout=timeout)
        response.raise_for_status()
        return response.json()
```

#### JavaScript / TypeScript
- Prefer TypeScript over JS for any non-trivial project
- Use `const`/`let`, never `var`
- Prefer `async/await` over `.then()` chains
- Use optional chaining `?.` and nullish coalescing `??`
- Type all function params and returns in TS

```typescript
// Good
async function fetchUser(id: string): Promise<User | null> {
  try {
    const res = await fetch(`/api/users/${id}`);
    if (!res.ok) return null;
    return res.json() as User;
  } catch {
    return null;
  }
}
```

#### React
- Functional components only (no class components)
- Custom hooks for reusable logic
- `useMemo`/`useCallback` only when profiling shows need
- Keys must be stable and unique (not array index)
- Co-locate state as close as possible to where it's used

#### SQL
- Always parameterize queries (never string concat)
- Use explicit column names, never `SELECT *`
- Add indexes for columns used in WHERE/JOIN
- Use CTEs for complex queries

---

## Debugging Workflow
1. **Reproduce** — minimal reproducible example
2. **Isolate** — binary search the problem
3. **Hypothesize** — form a theory before changing code
4. **Verify** — confirm fix doesn't break other things
5. **Document** — comment why the fix was needed

---

## Code Review Checklist
- [ ] Does it handle `null`/`undefined`/empty inputs?
- [ ] Are all errors caught and handled meaningfully?
- [ ] Are there any N+1 query problems?
- [ ] Is sensitive data logged anywhere?
- [ ] Are all external inputs validated/sanitized?
- [ ] Are async operations awaited?
- [ ] Are there memory leaks (event listeners, subscriptions not cleaned up)?

---

## File Generation Rules
- For code > 20 lines → always create a file artifact
- Include a docstring/comment block at the top of every file
- Include `if __name__ == "__main__":` guards in Python scripts
- Include `package.json` or `requirements.txt` when relevant
- Always show how to run the code

---

## Refactoring Patterns

### Extract Function
When a block of code does one clear thing → extract it.

### Replace Magic Numbers
```python
# Bad
if age > 18:
# Good
LEGAL_AGE = 18
if age > LEGAL_AGE:
```

### Early Return
```python
# Bad
def process(data):
    if data:
        if validate(data):
            return transform(data)
# Good
def process(data):
    if not data: return None
    if not validate(data): return None
    return transform(data)
```

---

## Common Patterns

### Retry with Backoff
```python
import time, random
from functools import wraps

def retry(max_attempts=3, base_delay=1.0, exceptions=(Exception,)):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts - 1:
                        raise
                    delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
                    time.sleep(delay)
        return wrapper
    return decorator
```

### Singleton
```python
class Singleton:
    _instance = None
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

### Observer / Event Bus
```python
from collections import defaultdict
from typing import Callable

class EventBus:
    def __init__(self):
        self._listeners: dict[str, list[Callable]] = defaultdict(list)

    def on(self, event: str, callback: Callable):
        self._listeners[event].append(callback)

    def emit(self, event: str, *args, **kwargs):
        for cb in self._listeners[event]:
            cb(*args, **kwargs)
```

---

## Performance Tips
- Profile before optimizing (`cProfile`, Chrome DevTools)
- Use generators for large sequences
- Batch database operations
- Cache expensive computations (`functools.lru_cache`)
- Use connection pooling for databases
- Lazy-load heavy modules

---

## Security Checklist
- Never hardcode secrets → use env vars or secret managers
- Validate all user input server-side
- Use parameterized SQL queries
- Set appropriate CORS headers
- Use HTTPS everywhere
- Rate-limit API endpoints
- Hash passwords with bcrypt/argon2, never MD5/SHA1

---

## Output Format
When generating code files, always:
1. Start with a brief description comment
2. Show imports at the top
3. Define constants/config before logic
4. Put main logic in functions/classes
5. End with usage example or `__main__` block
