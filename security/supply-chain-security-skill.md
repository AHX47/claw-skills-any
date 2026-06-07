# Supply Chain Security — Defensive Skills

## What Is a Supply Chain Attack?
An attacker compromises software/hardware *before* it reaches the end user — targeting build pipelines, open source packages, CI/CD systems, or hardware firmware to inject malicious code at scale.

Famous examples: SolarWinds (2020), XZ Utils (2024), event-stream npm package.

---

## Defending Your Dependencies

### Pin exact versions — never use wildcards
```toml
# pyproject.toml — BAD
dependencies = ["requests>=2.0"]

# GOOD — pin exact with hash verification
dependencies = ["requests==2.31.0"]
```

```bash
# Generate locked requirements with hashes
pip-compile --generate-hashes requirements.in > requirements.txt

# Install only from locked + verified hashes
pip install --require-hashes -r requirements.txt
```

### Verify package integrity
```bash
# npm — use package-lock.json and audit
npm ci                   # install from lock file exactly
npm audit                # check known vulnerabilities
npm audit fix            # auto-fix where safe

# Python
pip-audit                # scan for known CVEs
safety check -r requirements.txt

# Check PyPI package hash manually
pip download requests==2.31.0 --no-deps
sha256sum requests-2.31.0-*.whl
# Compare to published hash on PyPI
```

### Automated dependency scanning
```yaml
# GitHub Actions — Dependabot + audit
name: Security
on: [push, pull_request, schedule: [{cron: "0 6 * * 1"}]]
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Python audit
        run: pip install pip-audit && pip-audit -r requirements.txt
      - name: Node audit
        run: npm audit --audit-level=moderate
      - uses: github/codeql-action/analyze@v3
```

---

## Securing Your Build Pipeline

### Principle: least privilege for CI/CD
```yaml
# GitHub Actions — minimal permissions
permissions:
  contents: read        # not write unless needed
  packages: write       # only for publish job
  security-events: write

# Use OIDC tokens instead of long-lived secrets
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123:role/GitHubDeployRole
    aws-region: us-east-1
    # No static AWS keys — token is ephemeral
```

### Signed commits and tags
```bash
# Set up GPG signing
git config --global commit.gpgsign true
git config --global user.signingkey YOUR_GPG_KEY_ID

# Verify a commit
git verify-commit abc123

# Sign a release tag
git tag -s v1.0.0 -m "Release 1.0.0"
git verify-tag v1.0.0
```

### Reproducible builds
```dockerfile
# Pin base image by digest — not just tag
FROM python:3.12.3-slim@sha256:a1b2c3d4e5f6...

# Use --no-cache and exact versions
RUN pip install --no-cache-dir \
    requests==2.31.0 \
    fastapi==0.111.0
```

---

## Monitoring & Detection

### Detect unexpected outbound connections
```bash
# Monitor process network connections
ss -tulpn | grep python
netstat -an | grep ESTABLISHED

# Audit DNS queries (detect C2 beacons)
tcpdump -i any port 53 -n

# Check for unexpected cron jobs
crontab -l
cat /etc/cron.d/*
systemctl list-timers
```

### File integrity monitoring
```bash
# Create baseline hashes of critical files
sha256sum /usr/local/bin/python3 > /etc/file-hashes.txt
sha256sum /opt/myapp/**/*.py     >> /etc/file-hashes.txt

# Verify against baseline
sha256sum --check /etc/file-hashes.txt

# AIDE (Advanced Intrusion Detection Environment)
aide --init    # create database
aide --check   # check for changes
```

### Detect typosquatting packages
```bash
# Check for packages with similar names to popular ones
# requests vs. request, numpy vs. numpyy
pip search requests  # check exact spelling
# Always verify on pypi.org/project/<name> before installing
```

---

## Secure Development Practices

### Code review gates
```yaml
# Branch protection rules (GitHub)
# - Require 2 approvals for main
# - Require status checks: tests, audit, lint
# - Require signed commits
# - No force push to main/production
```

### SBOM (Software Bill of Materials)
```bash
# Generate SBOM for your project
pip install cyclonedx-bom
cyclonedx-py -r requirements.txt -o sbom.json

# For containers
syft my-image:latest -o json > container-sbom.json
```

---

## Incident Response for Compromised Dependency
1. **Isolate** — remove the compromised package immediately
2. **Assess** — what did the package have access to? (env vars, files, network)
3. **Rotate** — all secrets/tokens that were in scope
4. **Audit** — check logs for unusual outbound connections or data exfiltration
5. **Notify** — inform security team and affected users if data was exposed
6. **Report** — file CVE if you discovered the vulnerability

---

## Checklist
- [ ] All dependencies pinned to exact versions with hashes
- [ ] Automated vulnerability scanning on every PR
- [ ] CI/CD uses minimal permissions + ephemeral credentials
- [ ] Release tags signed with GPG
- [ ] Docker images pinned by digest
- [ ] SBOM generated for each release
- [ ] Alerting on unexpected outbound network connections
- [ ] File integrity monitoring on production systems
- [ ] Dependency review process for new packages
- [ ] Private package registry mirrors for critical deps
