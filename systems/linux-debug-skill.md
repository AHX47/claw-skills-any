# Linux + CMD + Debug + Sandbox Skills

## Essential Linux Commands
```bash
# File system navigation
ls -lah --color=auto          # list with human sizes + hidden
find . -name "*.py" -mtime -7 # files modified last 7 days
find . -size +10M -type f     # files larger than 10MB
du -sh */ | sort -hr           # directory sizes sorted
df -h                          # disk usage

# Text processing
grep -rn "TODO" src/           # recursive search with line numbers
grep -E "error|warn" log.txt   # regex search
awk '{print $1,$3}' file.txt   # print columns 1 and 3
sed 's/old/new/g' file.txt     # replace in file
sort file.txt | uniq -c | sort -nr  # count unique lines
cut -d',' -f2,4 data.csv       # cut CSV columns

# Process management
ps aux | grep python           # find processes
htop                           # interactive process viewer
kill -9 PID                    # force kill
pkill -f "main.py"             # kill by name pattern
lsof -i :8080                  # what's using port 8080
nohup python main.py &         # run in background, persist after logout
systemctl status myapp         # service status
journalctl -u myapp -f         # follow service logs

# Network
curl -s https://api.example.com/users | python3 -m json.tool
wget -c https://example.com/file.zip  # resumable download
netstat -tulpn | grep LISTEN   # listening ports
ss -s                           # socket statistics
tcpdump -i any -n port 80      # capture HTTP traffic
ping -c 4 8.8.8.8              # connectivity test
traceroute google.com           # route tracing
nmap -sV localhost              # port scan + service detection

# Permissions
chmod 755 script.sh            # rwxr-xr-x
chmod +x script.sh             # add execute
chown user:group file          # change owner
sudo -u www-data python run.py # run as different user

# Monitoring
tail -f /var/log/syslog        # follow system log
watch -n 2 df -h               # refresh df every 2s
vmstat 1                       # VM stats every second
iostat -x 1                    # I/O stats
```

## Bash Scripting Best Practices
```bash
#!/usr/bin/env bash
set -euo pipefail  # exit on error, unbound var, pipe failure
IFS=$'\n\t'        # safer word splitting

# Variables
readonly APP_DIR="/opt/myapp"
readonly LOG_FILE="/var/log/myapp.log"
readonly TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Functions
log()  { echo "[$(date +'%H:%M:%S')] $*" | tee -a "$LOG_FILE"; }
die()  { log "ERROR: $*"; exit 1; }
usage(){ echo "Usage: $0 <command>"; exit 0; }

# Argument parsing
[[ $# -eq 0 ]] && usage
case "$1" in
    start)   start_app ;;
    stop)    stop_app  ;;
    restart) stop_app; start_app ;;
    *)       die "Unknown command: $1" ;;
esac

# Safe file operations
tmpfile=$(mktemp) || die "Cannot create temp file"
trap "rm -f $tmpfile" EXIT  # cleanup on exit

# Check command exists
command -v python3 >/dev/null 2>&1 || die "python3 not installed"

# Loop with progress
total=100
for i in $(seq 1 $total); do
    process_item "$i"
    printf "\rProgress: %d/%d" "$i" "$total"
done
echo ""  # newline after progress
```

## Python Sandbox (safe execution)
```python
import subprocess, sys, tempfile, os
from pathlib import Path

def run_code_sandboxed(code: str, timeout: int = 10,
                         max_output: int = 10000) -> dict:
    """Execute Python code in isolated subprocess."""
    with tempfile.NamedTemporaryFile(suffix=".py", mode="w",
                                      delete=False, encoding="utf-8") as f:
        f.write(code); tmp = f.name
    try:
        result = subprocess.run(
            [sys.executable, "-c",
             f"import resource; resource.setrlimit(resource.RLIMIT_AS,(256*1024*1024,-1)); exec(open('{tmp}').read())"],
            capture_output=True, text=True,
            timeout=timeout, check=False,
            env={**os.environ, "PYTHONDONTWRITEBYTECODE":"1"},
        )
        return {
            "stdout":    result.stdout[:max_output],
            "stderr":    result.stderr[:2000],
            "returncode":result.returncode,
            "ok":        result.returncode == 0,
        }
    except subprocess.TimeoutExpired:
        return {"stdout":"", "stderr":"Timeout exceeded", "returncode":-1, "ok":False}
    finally:
        os.unlink(tmp)
```

## Debugging Skills
```python
# Python debugger
import pdb, ipdb

def buggy_function(data):
    result = []
    for item in data:
        pdb.set_trace()  # breakpoint — use 'n' next, 's' step, 'c' continue, 'p var' print
        processed = transform(item)
        result.append(processed)
    return result

# Built-in breakpoint (Python 3.7+)
def another_fn(x):
    breakpoint()  # same as pdb.set_trace()
    return x * 2

# Rich traceback (much more readable)
from rich.traceback import install
install(show_locals=True)  # add at top of main.py

# Logging levels for debugging
import logging
logging.basicConfig(level=logging.DEBUG,
                     format="%(asctime)s %(name)s %(levelname)s %(message)s")
log = logging.getLogger(__name__)
log.debug(f"Processing {item!r}")   # use !r for safe repr
log.info("Step 2 complete")
log.warning("Cache miss for %s", key)
log.error("Failed to connect: %s", exc, exc_info=True)  # logs stack trace
```

## Docker Debugging
```bash
# Get shell in running container
docker exec -it container_name bash

# Get shell in new container from same image
docker run --rm -it --entrypoint bash myimage:latest

# Inspect container
docker inspect container_name | python3 -m json.tool
docker logs container_name --tail 100 -f

# Copy files in/out
docker cp container_name:/app/log.txt ./log.txt
docker cp ./config.json container_name:/app/

# Resource usage
docker stats --no-stream

# Debug network
docker network ls
docker network inspect bridge
```

## GUI & CLI Design
```python
# Rich CLI with colors, tables, progress
from rich.console import Console
from rich.table   import Table
from rich.progress import Progress, SpinnerColumn, TextColumn
from rich import print as rprint

console = Console()

def show_table(data: list[dict]):
    table = Table(title="Results", style="cyan")
    for col in data[0].keys():
        table.add_column(col, style="white", no_wrap=True)
    for row in data:
        table.add_row(*[str(v) for v in row.values()])
    console.print(table)

def run_with_progress(tasks):
    with Progress(SpinnerColumn(), TextColumn("{task.description}"),
                   transient=True) as progress:
        task = progress.add_task("Processing...", total=len(tasks))
        for t in tasks:
            process(t)
            progress.advance(task)

# Argparse CLI
import argparse
parser = argparse.ArgumentParser(description="My Tool")
parser.add_argument("command",  choices=["run","build","test"])
parser.add_argument("--config", default="config.json")
parser.add_argument("--verbose", "-v", action="store_true")
parser.add_argument("--port",   type=int, default=8080)
args = parser.parse_args()
```
