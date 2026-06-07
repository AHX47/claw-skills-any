# Security Research Skills — Ethical & Defensive

> This skill covers **defensive security**: understanding how vulnerabilities work in order to build better defenses, conduct authorized security assessments, and contribute to responsible disclosure programs.

## Vulnerability Research Mindset
```
Before any security testing:
✅ Written authorization (scope, dates, authorized by owner)
✅ Bug bounty program rules reviewed
✅ Testing environment isolated from production
✅ No data exfiltration — report existence only
✅ Responsible disclosure timeline honored
```

## Common Vulnerability Classes (to defend against)

### Injection Flaws
```python
# SQL Injection — defend with parameterized queries (see database-security-skill.md)
# Command Injection — defend with:
import shlex, subprocess

def safe_run(user_input: str):
    # NEVER: os.system(f"ping {user_input}")
    # GOOD: validate + use argument list
    hostname = user_input.strip()
    if not re.match(r'^[a-zA-Z0-9.\-]+$', hostname):
        raise ValueError("Invalid hostname")
    result = subprocess.run(["ping", "-c", "3", hostname],
                             capture_output=True, text=True, timeout=10)
    return result.stdout

# Path Traversal — defend with:
from pathlib import Path

BASE_DIR = Path("/var/uploads").resolve()

def safe_read(filename: str) -> bytes:
    safe_path = (BASE_DIR / filename).resolve()
    if not str(safe_path).startswith(str(BASE_DIR)):
        raise PermissionError("Path traversal detected")
    return safe_path.read_bytes()
```

### Authentication Flaws
```python
# Timing-safe comparison (prevent timing attacks on secrets)
import hmac

def verify_api_key(provided: str, stored: str) -> bool:
    # NEVER: return provided == stored  (timing leak)
    # GOOD: constant-time comparison
    return hmac.compare_digest(provided.encode(), stored.encode())

# JWT algorithm confusion attack — defense:
import jwt
# ALWAYS specify algorithm explicitly
payload = jwt.decode(token, PUBLIC_KEY, algorithms=["RS256"])  # not ["RS256","HS256"]

# CSRF protection
import secrets
def generate_csrf_token() -> str:
    return secrets.token_hex(32)  # cryptographically random
```

### SSRF (Server-Side Request Forgery) Defense
```python
import ipaddress, socket
from urllib.parse import urlparse

BLOCKED_HOSTS = {"169.254.169.254", "metadata.google.internal"}  # cloud metadata
BLOCKED_NETS  = [ipaddress.ip_network("10.0.0.0/8"),
                  ipaddress.ip_network("172.16.0.0/12"),
                  ipaddress.ip_network("192.168.0.0/16"),
                  ipaddress.ip_network("127.0.0.0/8")]

def is_safe_url(url: str) -> bool:
    parsed = urlparse(url)
    if parsed.scheme not in ("http","https"): return False
    try:
        ip = socket.gethostbyname(parsed.hostname)
        ip_obj = ipaddress.ip_address(ip)
        if any(ip_obj in net for net in BLOCKED_NETS): return False
        if ip in BLOCKED_HOSTS: return False
    except Exception:
        return False
    return True
```

## Static Analysis Setup
```bash
# Python security scanning
pip install bandit semgrep safety
bandit -r src/ -ll              # report medium+ issues
semgrep --config=p/python src/  # OWASP rules
safety check -r requirements.txt # known CVE check

# JavaScript
npm install -g retire
retire --path ./           # check for vulnerable libraries
npx snyk test              # comprehensive JS/TS scan

# Secrets scanning
pip install detect-secrets
detect-secrets scan --baseline .secrets.baseline
detect-secrets audit .secrets.baseline

# Git pre-commit hook
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash
detect-secrets-hook --baseline .secrets.baseline $(git diff --cached --name-only)
EOF
chmod +x .git/hooks/pre-commit
```

## Penetration Testing Methodology (Authorized)
```
Phase 1 — Reconnaissance (passive)
  - Review public documentation, job postings, GitHub
  - DNS enumeration, certificate transparency (crt.sh)
  - No active scanning yet

Phase 2 — Scanning (active — requires auth)
  - Port scan: nmap -sV -sC target
  - Web scan: nikto, gobuster, feroxbuster
  - Identify tech stack, versions

Phase 3 — Vulnerability Assessment
  - Map findings to CVE database
  - Manual verification of each finding
  - OWASP Top 10 manual checks

Phase 4 — Exploitation (authorized scope only)
  - Prove impact — no unnecessary data access
  - Document exact steps to reproduce
  - Stop at proof-of-concept, don't escalate unnecessarily

Phase 5 — Reporting
  - Executive summary (business risk)
  - Technical detail (reproduction steps)
  - Severity rating (CVSS score)
  - Remediation recommendations
  - Evidence screenshots/logs
```

## Responsible Disclosure Process
```
1. Discover vulnerability
2. Document: steps to reproduce, impact, proof
3. Identify security contact: security.txt, HackerOne, Bugcrowd, email
4. Report privately with 90-day disclosure deadline (standard)
5. Coordinate with vendor on patch timeline
6. If no response after 90 days: consider limited public disclosure
7. Never: publish exploits before patch, sell to third parties
```

## Hardening Checklist
```bash
# Linux server hardening
# 1. Disable root SSH
sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
# 2. SSH key auth only
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
# 3. Fail2ban
apt install fail2ban && systemctl enable fail2ban
# 4. UFW firewall
ufw default deny incoming; ufw allow ssh; ufw allow 443; ufw enable
# 5. Auto security updates
apt install unattended-upgrades && dpkg-reconfigure -plow unattended-upgrades
# 6. Remove unnecessary services
systemctl disable avahi-daemon cups bluetooth
```

## Security Headers (Web)
```python
# FastAPI security headers middleware
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        response.headers["X-Content-Type-Options"]  = "nosniff"
        response.headers["X-Frame-Options"]          = "DENY"
        response.headers["X-XSS-Protection"]         = "1; mode=block"
        response.headers["Referrer-Policy"]           = "strict-origin-when-cross-origin"
        response.headers["Permissions-Policy"]        = "camera=(), microphone=(), geolocation=()"
        response.headers["Content-Security-Policy"]   = (
            "default-src 'self'; "
            "script-src 'self' 'nonce-{nonce}'; "
            "style-src 'self' https://fonts.googleapis.com; "
            "img-src 'self' data: https:; "
            "connect-src 'self' wss:; "
            "frame-ancestors 'none'"
        )
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains; preload"
        return response
```
