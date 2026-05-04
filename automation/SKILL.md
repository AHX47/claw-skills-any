# Automation Skill — Claude Best Practices

## Overview
This skill guides Claude to build robust automation systems: scripts, pipelines, schedulers, file watchers, web scrapers, CI/CD helpers, and workflow automation tools.

Use when: automating repetitive tasks, building pipelines, scraping data, scheduling jobs, processing files, integrating APIs, or building bots.

---

## Core Automation Principles

1. **Idempotent** — running twice produces the same result as once
2. **Resumable** — can restart from where it failed
3. **Observable** — logs everything meaningful, silent on success
4. **Self-healing** — retries transient failures automatically
5. **Bounded** — never runs forever, always has timeouts
6. **Auditable** — record what changed and when

---

## File Automation

### Watch a directory for changes
```python
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
import time

class FileHandler(FileSystemEventHandler):
    def on_created(self, event):
        if not event.is_directory:
            print(f"New file: {event.src_path}")
            process_file(event.src_path)

    def on_modified(self, event):
        if not event.is_directory:
            print(f"Modified: {event.src_path}")

def watch(path: str):
    observer = Observer()
    observer.schedule(FileHandler(), path, recursive=True)
    observer.start()
    try:
        while True: time.sleep(1)
    finally:
        observer.stop(); observer.join()

# pip install watchdog
```

### Batch process files
```python
from pathlib import Path
from concurrent.futures import ThreadPoolExecutor, as_completed
import logging

def batch_process(input_dir: str, output_dir: str,
                   process_fn, pattern: str = "*",
                   workers: int = 4) -> dict:
    inputs  = list(Path(input_dir).glob(pattern))
    out_dir = Path(output_dir)
    out_dir.mkdir(parents=True, exist_ok=True)
    results = {"ok": 0, "fail": 0, "errors": []}

    with ThreadPoolExecutor(max_workers=workers) as pool:
        futures = {pool.submit(process_fn, f, out_dir): f for f in inputs}
        for future in as_completed(futures):
            src = futures[future]
            try:
                future.result()
                results["ok"] += 1
                logging.info(f"✓ {src.name}")
            except Exception as e:
                results["fail"] += 1
                results["errors"].append({"file": str(src), "error": str(e)})
                logging.error(f"✗ {src.name}: {e}")

    return results
```

### Safe file operations
```python
import shutil
from pathlib import Path
from datetime import datetime

def safe_write(path: str, content: str, backup: bool = True):
    """Write file with automatic backup of existing content."""
    p = Path(path)
    if p.exists() and backup:
        ts  = datetime.now().strftime("%Y%m%d_%H%M%S")
        bak = p.with_suffix(f".{ts}.bak")
        shutil.copy2(p, bak)
    p.parent.mkdir(parents=True, exist_ok=True)
    p.write_text(content, encoding="utf-8")
    return str(p)
```

---

## Web Scraping

### Polite scraper with rate limiting
```python
import httpx, time, random
from bs4 import BeautifulSoup

class PoliteScraper:
    def __init__(self, delay: float = 1.0, jitter: float = 0.5,
                  headers: dict = None):
        self.delay   = delay
        self.jitter  = jitter
        self.client  = httpx.Client(
            headers=headers or {
                "User-Agent": "Mozilla/5.0 (compatible; ResearchBot/1.0)"
            },
            follow_redirects=True,
            timeout=15,
        )
        self._last_req = 0

    def get(self, url: str) -> httpx.Response:
        # Rate limiting
        elapsed = time.time() - self._last_req
        wait    = self.delay + random.uniform(0, self.jitter)
        if elapsed < wait:
            time.sleep(wait - elapsed)
        resp = self.client.get(url)
        resp.raise_for_status()
        self._last_req = time.time()
        return resp

    def get_soup(self, url: str) -> BeautifulSoup:
        return BeautifulSoup(self.get(url).text, "html.parser")

    def __enter__(self): return self
    def __exit__(self, *_): self.client.close()
```

### Extract structured data
```python
def extract_table(soup: BeautifulSoup, table_index: int = 0) -> list[dict]:
    tables = soup.find_all("table")
    if not tables or table_index >= len(tables):
        return []
    table   = tables[table_index]
    headers = [th.get_text(strip=True) for th in table.find_all("th")]
    rows    = []
    for tr in table.find_all("tr")[1:]:
        cells = [td.get_text(strip=True) for td in tr.find_all("td")]
        if cells and len(cells) == len(headers):
            rows.append(dict(zip(headers, cells)))
    return rows
```

---

## Scheduling

### Simple cron-style scheduler
```python
import schedule, time, threading, logging

class Scheduler:
    def __init__(self):
        self._running = False
        self._thread  = None

    def every(self, interval: int, unit: str, job_fn, job_name: str = ""):
        """unit: 'seconds' | 'minutes' | 'hours' | 'days'"""
        getattr(schedule.every(interval), unit).do(self._safe_run, job_fn, job_name)

    def daily_at(self, time_str: str, job_fn, job_name: str = ""):
        """time_str: 'HH:MM'"""
        schedule.every().day.at(time_str).do(self._safe_run, job_fn, job_name)

    def _safe_run(self, fn, name: str):
        try:
            fn()
        except Exception as e:
            logging.error(f"Job '{name}' failed: {e}")

    def start(self, blocking: bool = False):
        self._running = True
        if blocking:
            while self._running:
                schedule.run_pending(); time.sleep(1)
        else:
            def _loop():
                while self._running:
                    schedule.run_pending(); time.sleep(1)
            self._thread = threading.Thread(target=_loop, daemon=True)
            self._thread.start()

    def stop(self): self._running = False

# pip install schedule
# Usage:
# s = Scheduler()
# s.every(30, "minutes", sync_data, "data-sync")
# s.daily_at("03:00", backup_db, "nightly-backup")
# s.start()
```

---

## API Integration

### Generic REST client with auth
```python
import httpx
from typing import Any

class APIClient:
    def __init__(self, base_url: str, api_key: str = None,
                  token: str = None, timeout: int = 30):
        headers = {"Content-Type": "application/json"}
        if api_key:   headers["X-API-Key"]     = api_key
        if token:     headers["Authorization"] = f"Bearer {token}"
        self.client  = httpx.Client(base_url=base_url,
                                     headers=headers, timeout=timeout)

    def get(self, path: str, params: dict = None) -> Any:
        r = self.client.get(path, params=params); r.raise_for_status()
        return r.json()

    def post(self, path: str, data: dict) -> Any:
        r = self.client.post(path, json=data); r.raise_for_status()
        return r.json()

    def put(self, path: str, data: dict) -> Any:
        r = self.client.put(path, json=data); r.raise_for_status()
        return r.json()

    def delete(self, path: str) -> bool:
        r = self.client.delete(path)
        return r.status_code in (200, 204)

    def __enter__(self): return self
    def __exit__(self, *_): self.client.close()
```

---

## Data Pipeline

### ETL pipeline skeleton
```python
from typing import Iterable, Callable, TypeVar
from dataclasses import dataclass, field
import logging, time

T = TypeVar("T")

@dataclass
class PipelineStats:
    processed: int = 0
    succeeded: int = 0
    failed:    int = 0
    duration:  float = 0.0

class Pipeline:
    def __init__(self, name: str):
        self.name   = name
        self.stages: list[tuple[str, Callable]] = []

    def add_stage(self, name: str, fn: Callable) -> "Pipeline":
        self.stages.append((name, fn)); return self

    def run(self, source: Iterable) -> PipelineStats:
        stats = PipelineStats()
        start = time.time()
        for item in source:
            stats.processed += 1
            try:
                result = item
                for stage_name, stage_fn in self.stages:
                    result = stage_fn(result)
                    if result is None:
                        logging.debug(f"Item filtered at stage '{stage_name}'")
                        break
                if result is not None:
                    stats.succeeded += 1
            except Exception as e:
                stats.failed += 1
                logging.error(f"Pipeline '{self.name}' error: {e}")
        stats.duration = time.time() - start
        logging.info(f"Pipeline done: {stats}")
        return stats

# Usage:
# pipe = Pipeline("user-import")
#     .add_stage("validate",  validate_user)
#     .add_stage("transform", normalize_user)
#     .add_stage("load",      save_to_db)
# stats = pipe.run(csv_rows)
```

---

## Subprocess & Shell Automation

```python
import subprocess
from pathlib import Path

def run_cmd(cmd: list[str] | str, cwd: str = None,
             timeout: int = 60, env: dict = None) -> tuple[int, str, str]:
    """Run a shell command safely. Returns (returncode, stdout, stderr)."""
    result = subprocess.run(
        cmd, shell=isinstance(cmd, str),
        capture_output=True, text=True,
        cwd=cwd, timeout=timeout, env=env,
    )
    return result.returncode, result.stdout.strip(), result.stderr.strip()

def run_script(script_path: str, args: list[str] = None,
               cwd: str = None) -> tuple[bool, str]:
    """Run a Python script. Returns (success, output)."""
    cmd  = ["python3", script_path] + (args or [])
    code, out, err = run_cmd(cmd, cwd=cwd or str(Path(script_path).parent))
    output = out + ("\n" + err if err else "")
    return code == 0, output
```

---

## Email Automation

```python
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text      import MIMEText
from email.mime.base      import MIMEBase
from email               import encoders
from pathlib              import Path

def send_email(smtp_host: str, smtp_port: int,
                sender: str, password: str,
                to: list[str], subject: str,
                body: str, html: str = None,
                attachments: list[str] = None):
    msg = MIMEMultipart("alternative")
    msg["Subject"] = subject
    msg["From"]    = sender
    msg["To"]      = ", ".join(to)
    msg.attach(MIMEText(body, "plain"))
    if html: msg.attach(MIMEText(html, "html"))
    for path in (attachments or []):
        p = Path(path)
        part = MIMEBase("application", "octet-stream")
        part.set_payload(p.read_bytes())
        encoders.encode_base64(part)
        part.add_header("Content-Disposition", f"attachment; filename={p.name}")
        msg.attach(part)
    with smtplib.SMTP_SSL(smtp_host, smtp_port) as s:
        s.login(sender, password)
        s.sendmail(sender, to, msg.as_string())
```

---

## Logging Best Practices

```python
import logging, sys
from pathlib import Path
from datetime import datetime

def setup_logging(name: str, log_dir: str = "logs",
                   level: int = logging.INFO) -> logging.Logger:
    Path(log_dir).mkdir(exist_ok=True)
    log_file = Path(log_dir) / f"{name}_{datetime.now():%Y%m%d}.log"
    fmt = "%(asctime)s [%(levelname)s] %(name)s: %(message)s"
    logging.basicConfig(
        level=level, format=fmt,
        handlers=[
            logging.FileHandler(log_file, encoding="utf-8"),
            logging.StreamHandler(sys.stdout),
        ]
    )
    return logging.getLogger(name)
```

---

## Automation Checklist
- [ ] Does it handle partial failures gracefully?
- [ ] Can it be run multiple times safely (idempotent)?
- [ ] Does it log enough to debug issues later?
- [ ] Are there timeouts on all network/subprocess calls?
- [ ] Is it storing credentials securely (env vars, not hardcoded)?
- [ ] Does it send alerts on failure?
- [ ] Is there a dry-run mode before making real changes?
- [ ] Are file operations atomic (write to temp, then rename)?
