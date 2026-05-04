# Coding Patterns Reference

## Python Patterns

### Context Manager
```python
from contextlib import contextmanager

@contextmanager
def managed_resource(config):
    resource = acquire(config)
    try:
        yield resource
    finally:
        release(resource)

# Usage: with managed_resource(cfg) as r: use(r)
```

### Dataclass with validation
```python
from dataclasses import dataclass, field
from typing import ClassVar

@dataclass
class Config:
    host:    str   = "localhost"
    port:    int   = 8080
    debug:   bool  = False
    tags:    list  = field(default_factory=list)

    VALID_PORTS: ClassVar[range] = range(1, 65536)

    def __post_init__(self):
        if self.port not in self.VALID_PORTS:
            raise ValueError(f"Invalid port: {self.port}")
```

### Functional pipeline
```python
from functools import reduce
from typing import Callable, TypeVar

T = TypeVar("T")

def pipe(*fns: Callable) -> Callable:
    """Compose functions left-to-right."""
    return lambda x: reduce(lambda v, f: f(v), fns, x)

# Usage:
# process = pipe(clean, validate, transform, save)
# result  = process(raw_data)
```

### Result type (no exceptions)
```python
from dataclasses import dataclass
from typing import Generic, TypeVar

T = TypeVar("T")

@dataclass
class Ok(Generic[T]):
    value: T
    ok: bool = True

@dataclass
class Err:
    error: str
    ok: bool = False

Result = Ok | Err

def divide(a: float, b: float) -> Result:
    if b == 0: return Err("Division by zero")
    return Ok(a / b)

# Usage:
# r = divide(10, 2)
# if r.ok: print(r.value)
# else:    print(r.error)
```

---

## JavaScript / TypeScript Patterns

### Async queue with concurrency
```typescript
class AsyncQueue<T> {
  private queue: Array<() => Promise<T>> = [];
  private running = 0;
  private concurrency: number;

  constructor(concurrency = 3) {
    this.concurrency = concurrency;
  }

  add(task: () => Promise<T>): Promise<T> {
    return new Promise((resolve, reject) => {
      this.queue.push(async () => {
        try   { resolve(await task()); }
        catch { reject; }
      });
      this.run();
    });
  }

  private async run() {
    if (this.running >= this.concurrency || !this.queue.length) return;
    this.running++;
    const task = this.queue.shift()!;
    await task();
    this.running--;
    this.run();
  }
}
```

### Event emitter (typed)
```typescript
type Listener<T> = (data: T) => void;

class TypedEmitter<Events extends Record<string, any>> {
  private listeners = new Map<keyof Events, Set<Listener<any>>>();

  on<K extends keyof Events>(event: K, fn: Listener<Events[K]>) {
    if (!this.listeners.has(event)) this.listeners.set(event, new Set());
    this.listeners.get(event)!.add(fn);
    return () => this.off(event, fn);
  }

  off<K extends keyof Events>(event: K, fn: Listener<Events[K]>) {
    this.listeners.get(event)?.delete(fn);
  }

  emit<K extends keyof Events>(event: K, data: Events[K]) {
    this.listeners.get(event)?.forEach(fn => fn(data));
  }
}

// Usage:
// interface Events { "user:login": { id: string }; "error": Error }
// const bus = new TypedEmitter<Events>();
// bus.on("user:login", ({ id }) => console.log(id));
```

---

## React Patterns

### Custom data fetching hook
```tsx
import { useState, useEffect, useCallback, useRef } from "react";

interface FetchState<T> {
  data:    T | null;
  loading: boolean;
  error:   string | null;
  refetch: () => void;
}

function useFetch<T>(url: string, deps: any[] = []): FetchState<T> {
  const [data,    setData]    = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error,   setError]   = useState<string | null>(null);
  const abortRef = useRef<AbortController>();

  const fetch_ = useCallback(async () => {
    abortRef.current?.abort();
    abortRef.current = new AbortController();
    setLoading(true); setError(null);
    try {
      const res = await fetch(url, { signal: abortRef.current.signal });
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      setData(await res.json());
    } catch (e: any) {
      if (e.name !== "AbortError") setError(e.message);
    } finally {
      setLoading(false);
    }
  }, [url, ...deps]);

  useEffect(() => { fetch_(); return () => abortRef.current?.abort(); }, [fetch_]);
  return { data, loading, error, refetch: fetch_ };
}
```

### Form state hook
```tsx
import { useState } from "react";

function useForm<T extends Record<string, any>>(initial: T) {
  const [values,  setValues]  = useState<T>(initial);
  const [errors,  setErrors]  = useState<Partial<Record<keyof T, string>>>({});
  const [touched, setTouched] = useState<Partial<Record<keyof T, boolean>>>({});

  const set = (field: keyof T, value: any) =>
    setValues(v => ({ ...v, [field]: value }));

  const touch = (field: keyof T) =>
    setTouched(t => ({ ...t, [field]: true }));

  const setError = (field: keyof T, msg: string) =>
    setErrors(e => ({ ...e, [field]: msg }));

  const reset = () => { setValues(initial); setErrors({}); setTouched({}); };

  const isValid = Object.keys(errors).length === 0;

  return { values, errors, touched, set, touch, setError, reset, isValid };
}
```

---

## Database Patterns

### Repository pattern
```python
from abc import ABC, abstractmethod
from typing import TypeVar, Generic, Optional

T = TypeVar("T")

class Repository(ABC, Generic[T]):
    @abstractmethod
    def find_by_id(self, id: int) -> Optional[T]: ...
    @abstractmethod
    def find_all(self) -> list[T]: ...
    @abstractmethod
    def save(self, entity: T) -> T: ...
    @abstractmethod
    def delete(self, id: int) -> bool: ...

class SQLiteUserRepo(Repository):
    def __init__(self, conn):
        self.conn = conn

    def find_by_id(self, id: int):
        row = self.conn.execute(
            "SELECT * FROM users WHERE id = ?", (id,)
        ).fetchone()
        return dict(row) if row else None

    def save(self, user: dict) -> dict:
        if user.get("id"):
            self.conn.execute(
                "UPDATE users SET name=?, email=? WHERE id=?",
                (user["name"], user["email"], user["id"])
            )
        else:
            cur = self.conn.execute(
                "INSERT INTO users (name, email) VALUES (?, ?)",
                (user["name"], user["email"])
            )
            user["id"] = cur.lastrowid
        self.conn.commit()
        return user
```

---

## Testing Patterns

### Parametrized tests (pytest)
```python
import pytest

@pytest.mark.parametrize("input,expected", [
    ("hello world", "Hello World"),
    ("",            ""),
    ("abc",         "Abc"),
])
def test_title_case(input, expected):
    assert title_case(input) == expected
```

### Mock HTTP (pytest)
```python
import pytest, responses, requests

@responses.activate
def test_api_call():
    responses.add(responses.GET, "https://api.example.com/users/1",
                  json={"id": 1, "name": "Alice"}, status=200)
    result = get_user(1)
    assert result["name"] == "Alice"
    assert len(responses.calls) == 1
```

### Fixture factory
```python
import pytest
from dataclasses import dataclass

@dataclass
class User:
    id: int; name: str; email: str; role: str = "user"

@pytest.fixture
def make_user():
    counter = {"n": 0}
    def factory(**kwargs) -> User:
        counter["n"] += 1
        n = counter["n"]
        return User(id=n, name=f"User {n}", email=f"user{n}@test.com", **kwargs)
    return factory

def test_admin(make_user):
    admin = make_user(role="admin")
    assert can_delete(admin) is True
```
