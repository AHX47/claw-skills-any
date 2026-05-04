# Automation Recipes

## GitHub Actions CI/CD

### Python CI workflow
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: "pip"
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: ruff check .
      - run: mypy .
      - run: pytest --cov=src --cov-report=xml -q
      - uses: codecov/codecov-action@v4
```

### Auto-deploy on tag
```yaml
name: Deploy
on:
  push:
    tags: ["v*"]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: docker build -t app:${{ github.ref_name }} .
      - name: Push to registry
        run: |
          echo ${{ secrets.REGISTRY_PASSWORD }} | docker login -u ${{ secrets.REGISTRY_USER }} --password-stdin
          docker push app:${{ github.ref_name }}
      - name: Deploy
        run: ssh deploy@${{ secrets.SERVER }} "cd /app && ./deploy.sh ${{ github.ref_name }}"
```

---

## Docker Automation

### Dockerfile best practices
```dockerfile
# Multi-stage build for Python
FROM python:3.12-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
# Copy only installed packages
COPY --from=builder /root/.local /root/.local
COPY src/ src/
ENV PATH=/root/.local/bin:$PATH \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1
# Run as non-root
RUN adduser --disabled-password --gecos "" appuser
USER appuser
CMD ["python", "-m", "src.main"]
```

### docker-compose for development
```yaml
# docker-compose.yml
version: "3.9"
services:
  app:
    build: .
    ports: ["8000:8000"]
    volumes: ["./src:/app/src"]   # hot reload
    environment:
      DATABASE_URL: postgres://postgres:pass@db:5432/appdb
      REDIS_URL: redis://redis:6379
    depends_on:
      db:    { condition: service_healthy }
      redis: { condition: service_started }
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: appdb
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s; timeout: 5s; retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru

volumes:
  pgdata:
```

---

## Makefile Automation

```makefile
.PHONY: install dev test lint format clean build docker-up docker-down

PYTHON   = python3
PIP      = $(PYTHON) -m pip
APP_NAME = myapp

install:
	$(PIP) install -r requirements.txt

dev:
	$(PIP) install -r requirements.txt -r requirements-dev.txt
	pre-commit install

test:
	pytest tests/ -v --cov=$(APP_NAME) --cov-report=term-missing

lint:
	ruff check .
	mypy src/

format:
	ruff format .
	ruff check --fix .

clean:
	find . -type d -name __pycache__ -exec rm -rf {} +
	find . -name "*.pyc" -delete
	rm -rf .pytest_cache .coverage dist build *.egg-info

build: clean
	$(PYTHON) -m build

docker-up:
	docker compose up -d

docker-down:
	docker compose down

db-migrate:
	alembic upgrade head

db-rollback:
	alembic downgrade -1
```

---

## Cron / Task Scheduling

### Linux crontab patterns
```bash
# Edit: crontab -e

# Every 5 minutes
*/5 * * * * /usr/bin/python3 /home/user/scripts/sync.py >> /var/log/sync.log 2>&1

# Daily at 2:30 AM
30 2 * * * /home/user/scripts/backup.sh

# Every Monday at 8 AM
0 8 * * 1 /home/user/scripts/weekly_report.py

# First day of month
0 0 1 * * /home/user/scripts/monthly_cleanup.py

# Every weekday (Mon-Fri) at 9 AM
0 9 * * 1-5 /home/user/scripts/daily_standup.py

# Reboot startup
@reboot /home/user/scripts/start_services.sh
```

### Systemd service + timer
```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=MyApp Background Task
After=network.target

[Service]
Type=oneshot
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/venv/bin/python -m myapp.tasks.sync
EnvironmentFile=/opt/myapp/.env
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/myapp.timer
[Unit]
Description=Run MyApp every 15 minutes

[Timer]
OnBootSec=2min
OnUnitActiveSec=15min
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
systemctl enable --now myapp.timer
systemctl list-timers
journalctl -u myapp.service -f
```

---

## Python Automation Scripts

### Backup script
```python
#!/usr/bin/env python3
"""backup.py — Automated backup with rotation."""
import shutil, os, logging
from pathlib import Path
from datetime import datetime, timedelta

def backup(source: str, dest: str, keep_days: int = 7):
    src  = Path(source)
    root = Path(dest)
    ts   = datetime.now().strftime("%Y%m%d_%H%M%S")
    out  = root / f"backup_{ts}"
    root.mkdir(parents=True, exist_ok=True)
    shutil.copytree(src, out)
    logging.info(f"Backup created: {out}")
    # Rotate old backups
    cutoff = datetime.now() - timedelta(days=keep_days)
    for old in sorted(root.glob("backup_*")):
        try:
            ts_str = old.name.split("_", 1)[1]
            bk_ts  = datetime.strptime(ts_str, "%Y%m%d_%H%M%S")
            if bk_ts < cutoff:
                shutil.rmtree(old)
                logging.info(f"Removed old backup: {old}")
        except Exception: pass
    return str(out)

if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO,
                         format="%(asctime)s %(levelname)s %(message)s")
    backup("/opt/myapp/data", "/backups/myapp", keep_days=7)
```

### Monitor + alert script
```python
#!/usr/bin/env python3
"""monitor.py — Service health monitor."""
import httpx, time, smtplib, os, logging
from email.mime.text import MIMEText

CHECKS = [
    {"name": "API",      "url": "https://api.myapp.com/health"},
    {"name": "Frontend", "url": "https://myapp.com"},
]
ALERT_EMAIL = os.getenv("ALERT_EMAIL")
SMTP_HOST   = os.getenv("SMTP_HOST", "smtp.gmail.com")

def check(url: str, timeout: int = 10) -> tuple[bool, int, str]:
    try:
        r = httpx.get(url, timeout=timeout, follow_redirects=True)
        return r.status_code < 400, r.status_code, ""
    except Exception as e:
        return False, 0, str(e)

def alert(service: str, error: str):
    if not ALERT_EMAIL: return
    msg = MIMEText(f"🚨 {service} is DOWN\n\nError: {error}")
    msg["Subject"] = f"[ALERT] {service} down"
    msg["From"]    = ALERT_EMAIL
    msg["To"]      = ALERT_EMAIL
    with smtplib.SMTP_SSL(SMTP_HOST, 465) as s:
        s.login(ALERT_EMAIL, os.getenv("SMTP_PASSWORD", ""))
        s.send_message(msg)

if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    for svc in CHECKS:
        ok, code, err = check(svc["url"])
        if ok:
            logging.info(f"✅ {svc['name']} OK ({code})")
        else:
            logging.error(f"❌ {svc['name']} DOWN: {err or code}")
            alert(svc["name"], err or f"HTTP {code}")
```

---

## Pre-commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-merge-conflict
      - id: detect-private-key
      - id: no-commit-to-branch
        args: [--branch, main, --branch, production]

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.10.0
    hooks:
      - id: mypy
        additional_dependencies: [types-requests]
```
