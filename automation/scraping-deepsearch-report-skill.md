# Web Scraping, DeepSearch & Report Generation Skills

## Polite Web Scraper
```python
import httpx, time, random
from bs4 import BeautifulSoup
from urllib.parse import urljoin, urlparse

class WebScraper:
    def __init__(self, delay=1.5, timeout=15):
        self.delay  = delay
        self.client = httpx.Client(
            headers={"User-Agent": "Mozilla/5.0 (compatible; ResearchBot/1.0)"},
            follow_redirects=True, timeout=timeout)
        self._last = 0

    def get(self, url: str) -> BeautifulSoup | None:
        wait = self.delay + random.uniform(0, 0.8)
        elapsed = time.time() - self._last
        if elapsed < wait: time.sleep(wait - elapsed)
        try:
            r = self.client.get(url); r.raise_for_status()
            self._last = time.time()
            return BeautifulSoup(r.text, "html.parser")
        except Exception as e:
            print(f"Error {url}: {e}"); return None

    def get_links(self, soup: BeautifulSoup, base_url: str,
                   same_domain=True) -> list[str]:
        links = []
        domain = urlparse(base_url).netloc
        for a in soup.find_all("a", href=True):
            url = urljoin(base_url, a["href"])
            if same_domain and urlparse(url).netloc != domain: continue
            if url.startswith("http"): links.append(url)
        return list(set(links))

    def extract_article(self, soup: BeautifulSoup) -> dict:
        title = (soup.find("h1") or soup.find("title") or soup.new_tag("x")).get_text(strip=True)
        # Remove nav, footer, ads
        for tag in soup(["nav","footer","aside","script","style","[class*=ad]",
                          "[class*=menu]","[id*=nav]","[id*=footer]"]):
            tag.decompose()
        body = soup.find("article") or soup.find("main") or soup.find("body")
        text = body.get_text(separator="\n", strip=True) if body else ""
        # Clean whitespace
        lines = [l.strip() for l in text.splitlines() if len(l.strip()) > 20]
        return {"title": title, "text": "\n".join(lines)}
```

## DeepSearch — Multi-Source Research
```python
import anthropic, json
from dataclasses import dataclass, field

client = anthropic.Anthropic()

@dataclass
class SearchResult:
    query:   str
    sources: list[dict] = field(default_factory=list)
    summary: str = ""

def deep_search(topic: str, depth: int = 3) -> SearchResult:
    """Multi-hop research using Claude with web search tool."""
    result = SearchResult(query=topic)
    messages = [{"role":"user","content":
        f"Research this topic thoroughly. Search multiple angles, verify facts, "
        f"find recent information. Topic: {topic}"}]

    for _ in range(depth):
        resp = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2000,
            tools=[{"type":"web_search_20250305","name":"web_search"}],
            messages=messages,
        )
        if resp.stop_reason == "end_turn":
            result.summary = next((b.text for b in resp.content if b.type=="text"),"")
            break
        # Process tool use
        tool_results = []
        for block in resp.content:
            if block.type == "tool_use" and block.name == "web_search":
                # Simulated search result (real implementation uses actual search)
                tool_results.append({
                    "type":"tool_result","tool_use_id":block.id,
                    "content":f"Search results for: {block.input['query']}"
                })
                result.sources.append({"query":block.input["query"]})
        messages.append({"role":"assistant","content":resp.content})
        if tool_results:
            messages.append({"role":"user","content":tool_results})

    return result
```

## Markdown Report Generator
```python
from datetime import datetime
from pathlib import Path
import json

class ReportGenerator:
    def __init__(self, title: str):
        self.title     = title
        self.sections  = []
        self.timestamp = datetime.now()

    def add_section(self, heading: str, content: str):
        self.sections.append({"heading": heading, "content": content})
        return self

    def add_table(self, heading: str, headers: list[str], rows: list[list]):
        col_widths = [max(len(str(h)), max((len(str(r[i])) for r in rows), default=0))
                      for i, h in enumerate(headers)]
        header_row = "| " + " | ".join(h.ljust(col_widths[i]) for i,h in enumerate(headers)) + " |"
        sep_row    = "|" + "|".join("-"*(w+2) for w in col_widths) + "|"
        data_rows  = ["| " + " | ".join(str(r[i]).ljust(col_widths[i]) for i in range(len(headers))) + " |"
                      for r in rows]
        table = "\n".join([header_row, sep_row] + data_rows)
        self.sections.append({"heading": heading, "content": table})
        return self

    def add_code(self, heading: str, code: str, lang="python"):
        self.sections.append({"heading": heading,
                               "content": f"```{lang}\n{code}\n```"})
        return self

    def render(self) -> str:
        lines = [f"# {self.title}", "",
                 f"*Generated: {self.timestamp.strftime('%Y-%m-%d %H:%M')}*", ""]
        lines.append("## Table of Contents")
        for i,s in enumerate(self.sections,1):
            anchor = s["heading"].lower().replace(" ","-").replace("/","")
            lines.append(f"{i}. [{s['heading']}](#{anchor})")
        lines.append("")
        for s in self.sections:
            lines += [f"## {s['heading']}", "", s["content"], ""]
        return "\n".join(lines)

    def save(self, path: str) -> str:
        content = self.render()
        Path(path).write_text(content, encoding="utf-8")
        return path

# Usage
report = ReportGenerator("Network Analysis Report")
report.add_section("Overview", "Analysis of 192.168.0.0/24 network...")
report.add_table("Connected Devices",
    ["IP", "MAC", "Hostname", "Status"],
    [["192.168.0.1","AA:BB:CC:DD","router","online"],
     ["192.168.0.10","11:22:33:44","PC-Ali","online"]])
report.add_code("Sample Query", "SELECT * FROM devices WHERE status='online'", "sql")
report.save("report.md")
```

## Extract Content from Documents
```python
from pathlib import Path
import json, re

def extract_from_pdf(path: str) -> str:
    import pdfplumber
    text = []
    with pdfplumber.open(path) as pdf:
        for page in pdf.pages:
            t = page.extract_text()
            if t: text.append(t)
    return "\n\n".join(text)

def extract_from_docx(path: str) -> str:
    from docx import Document
    doc   = Document(path)
    parts = []
    for para in doc.paragraphs:
        if para.text.strip(): parts.append(para.text)
    for table in doc.tables:
        for row in table.rows:
            parts.append(" | ".join(c.text.strip() for c in row.cells))
    return "\n".join(parts)

def extract_from_xlsx(path: str) -> str:
    import openpyxl
    wb = openpyxl.load_workbook(path, read_only=True)
    rows = []
    for ws in wb.worksheets:
        rows.append(f"## Sheet: {ws.title}")
        for row in ws.iter_rows(values_only=True):
            rows.append(" | ".join(str(c) for c in row if c is not None))
    return "\n".join(rows)

def extract_any(path: str) -> str:
    suffix = Path(path).suffix.lower()
    extractors = {".pdf":extract_from_pdf, ".docx":extract_from_docx,
                  ".xlsx":extract_from_xlsx, ".xls":extract_from_xlsx,
                  ".txt":lambda p: Path(p).read_text(encoding="utf-8",errors="ignore"),
                  ".json":lambda p: json.dumps(json.loads(Path(p).read_text()),indent=2)}
    fn = extractors.get(suffix)
    if not fn: raise ValueError(f"Unsupported: {suffix}")
    return fn(path)
```
