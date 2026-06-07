# Python Skills — Comprehensive Read/Write Guide

## Reading Python Code
When reading Python: identify entry points (`if __name__=="__main__"`, `app.run()`, `uvicorn.run()`), follow imports top-down, understand class hierarchies via `__init__` and `__mro__`, trace data flow through function signatures with type hints.

## Writing Python — Standards
```python
# Always: type hints, docstrings, f-strings, pathlib, dataclasses
from __future__ import annotations
from pathlib import Path
from dataclasses import dataclass, field
from typing import Any

@dataclass
class Config:
    host: str = "localhost"
    port: int = 8080
    tags: list[str] = field(default_factory=list)

    def __post_init__(self):
        if not 1 <= self.port <= 65535:
            raise ValueError(f"Invalid port: {self.port}")
```

## File I/O
```python
from pathlib import Path
import json, csv, tomllib

# Text
text = Path("file.txt").read_text(encoding="utf-8")
Path("out.txt").write_text(content, encoding="utf-8")

# JSON
data = json.loads(Path("data.json").read_text())
Path("out.json").write_text(json.dumps(data, ensure_ascii=False, indent=2))

# CSV
import csv
with open("data.csv", newline="", encoding="utf-8") as f:
    rows = list(csv.DictReader(f))

# TOML (Python 3.11+)
with open("config.toml", "rb") as f:
    cfg = tomllib.load(f)
```

## Async Python
```python
import asyncio, httpx

async def fetch_all(urls: list[str]) -> list[dict]:
    async with httpx.AsyncClient() as client:
        tasks = [client.get(u) for u in urls]
        responses = await asyncio.gather(*tasks, return_exceptions=True)
        return [r.json() for r in responses if not isinstance(r, Exception)]

asyncio.run(fetch_all(["https://api.example.com/a", "https://api.example.com/b"]))
```

## Error Handling
```python
# Specific, never bare except
class AppError(Exception):
    def __init__(self, msg: str, code: int = 500):
        super().__init__(msg); self.code = code

def safe_divide(a: float, b: float) -> float:
    if b == 0: raise AppError("Division by zero", 400)
    return a / b

try:
    result = safe_divide(10, 0)
except AppError as e:
    print(f"[{e.code}] {e}")
except Exception as e:
    print(f"Unexpected: {e}")
    raise  # re-raise unexpected
```

## Decorators
```python
import functools, time, logging

def retry(attempts=3, delay=1.0, exceptions=(Exception,)):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            for i in range(attempts):
                try: return fn(*args, **kwargs)
                except exceptions as e:
                    if i == attempts-1: raise
                    time.sleep(delay * 2**i)
        return wrapper
    return decorator

def timer(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        t = time.perf_counter()
        result = fn(*args, **kwargs)
        logging.info(f"{fn.__name__} took {time.perf_counter()-t:.3f}s")
        return result
    return wrapper
```

## Generators & Itertools
```python
from itertools import islice, chain, groupby

def chunked(iterable, size):
    it = iter(iterable)
    while chunk := list(islice(it, size)):
        yield chunk

# Lazy file processing — memory efficient
def read_large_csv(path: str):
    with open(path, encoding="utf-8") as f:
        reader = csv.DictReader(f)
        yield from reader
```

## Context Managers
```python
from contextlib import contextmanager, suppress

@contextmanager
def timer_ctx(name: str):
    t = time.perf_counter()
    try: yield
    finally: print(f"{name}: {time.perf_counter()-t:.3f}s")

# Suppress specific errors
with suppress(FileNotFoundError):
    Path("optional.txt").unlink()
```

## Testing
```python
import pytest
from unittest.mock import patch, MagicMock

@pytest.fixture
def client(app):
    return app.test_client()

@pytest.mark.parametrize("a,b,expected", [(2,3,5),(0,0,0),(-1,1,0)])
def test_add(a, b, expected):
    assert add(a, b) == expected

def test_api_call():
    with patch("mymodule.httpx.get") as mock_get:
        mock_get.return_value = MagicMock(json=lambda: {"ok": True})
        result = fetch_data("http://test.com")
        assert result["ok"] is True
```

## Performance
```python
from functools import lru_cache
import cProfile, pstats

@lru_cache(maxsize=128)
def expensive(n: int) -> int:
    return sum(i**2 for i in range(n))

# Profile
with cProfile.Profile() as pr:
    expensive(10000)
stats = pstats.Stats(pr).sort_stats("cumulative")
stats.print_stats(10)
```

## Environment & Config
```python
import os
from dotenv import load_dotenv

load_dotenv()  # loads .env file

DB_URL    = os.environ["DATABASE_URL"]          # required — raises if missing
DEBUG     = os.getenv("DEBUG", "false").lower() == "true"
MAX_CONN  = int(os.getenv("MAX_CONNECTIONS", "10"))
```

## Packaging
```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.backends.legacy:build"

[project]
name = "mypackage"
version = "1.0.0"
requires-python = ">=3.10"
dependencies = ["httpx>=0.27", "pydantic>=2"]

[project.scripts]
myapp = "mypackage.cli:main"
```
