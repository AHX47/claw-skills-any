# Dataset Generation Skills

## Synthetic Data with LLM
```python
import anthropic, json, random
from pathlib import Path

client = anthropic.Anthropic()

def generate_classification_dataset(
    task: str, labels: list[str],
    samples_per_label: int = 100,
    language: str = "Arabic"
) -> list[dict]:
    dataset = []
    for label in labels:
        prompt = f"""Generate {samples_per_label} unique {language} text examples for:
Task: {task}
Label: {label}
Rules: diverse styles, lengths 10-100 words, realistic, no duplicates.
Output ONLY a JSON array: [{{"text":"...","label":"{label}"}}, ...]"""

        resp = client.messages.create(
            model="claude-sonnet-4-20250514", max_tokens=4000,
            messages=[{"role":"user","content":prompt}])
        try:
            chunk = json.loads(resp.content[0].text)
            dataset.extend(chunk)
        except json.JSONDecodeError:
            pass  # retry logic here
    random.shuffle(dataset)
    return dataset

# Usage
data = generate_classification_dataset(
    task="Sentiment analysis of product reviews",
    labels=["positive","negative","neutral"],
    samples_per_label=50, language="Arabic")
Path("dataset.json").write_text(json.dumps(data, ensure_ascii=False, indent=2))
```

## CSV/JSONL Dataset Builder
```python
import csv, json
from dataclasses import dataclass, asdict

@dataclass
class Sample:
    text:  str
    label: str
    split: str = "train"  # train/val/test
    id:    str = ""

def save_dataset(samples: list[Sample], base_path: str):
    # JSONL format (HuggingFace compatible)
    splits = {}
    for s in samples:
        splits.setdefault(s.split, []).append(asdict(s))

    for split, rows in splits.items():
        path = f"{base_path}_{split}.jsonl"
        with open(path, "w", encoding="utf-8") as f:
            for row in rows:
                f.write(json.dumps(row, ensure_ascii=False) + "\n")
        print(f"Saved {len(rows)} samples → {path}")

    # Also save as CSV
    all_path = f"{base_path}_all.csv"
    with open(all_path, "w", newline="", encoding="utf-8") as f:
        w = csv.DictWriter(f, fieldnames=["id","text","label","split"])
        w.writeheader()
        w.writerows([asdict(s) for s in samples])
```

## Data Augmentation
```python
import random, re

class TextAugmenter:
    def __init__(self, language="ar"):
        self.lang = language

    def augment(self, text: str, n: int = 3) -> list[str]:
        variants = []
        methods  = [self.swap_words, self.delete_random, self.repeat_char]
        for _ in range(n):
            fn = random.choice(methods)
            variants.append(fn(text))
        return variants

    def swap_words(self, text: str) -> str:
        words = text.split()
        if len(words) < 3: return text
        i, j = random.sample(range(len(words)), 2)
        words[i], words[j] = words[j], words[i]
        return " ".join(words)

    def delete_random(self, text: str, p=0.1) -> str:
        words = text.split()
        return " ".join(w for w in words if random.random() > p)

    def repeat_char(self, text: str) -> str:
        # e.g. "جميل" → "جميييل"
        chars = list(text)
        for i in range(len(chars)):
            if chars[i].isalpha() and random.random() < 0.05:
                chars[i] = chars[i] * random.randint(2, 3)
        return "".join(chars)

def back_translate(text: str, intermediate_lang="en") -> str:
    """Paraphrase by translating to another language and back."""
    # Requires translation API (Google Translate, DeepL, etc.)
    translated = translate(text, src="ar", tgt=intermediate_lang)
    return translate(translated, src=intermediate_lang, tgt="ar")
```

## Quality Filtering
```python
def filter_dataset(samples: list[dict], lang="ar") -> list[dict]:
    import re
    clean = []
    seen  = set()

    for s in samples:
        text = s.get("text","").strip()

        # Remove duplicates
        if text in seen: continue
        seen.add(text)

        # Length filter
        words = text.split()
        if not 5 <= len(words) <= 500: continue

        # Arabic text check
        arabic_chars = len(re.findall(r"[\u0600-\u06FF]", text))
        if lang == "ar" and arabic_chars / max(len(text),1) < 0.5: continue

        # No excessive punctuation/symbols
        punct_ratio = len(re.findall(r"[!@#$%^&*]{2,}", text)) / max(len(text),1)
        if punct_ratio > 0.1: continue

        clean.append(s)

    print(f"Filtered: {len(samples)} → {len(clean)} samples")
    return clean
```

## HuggingFace Dataset Upload
```python
from datasets import Dataset, DatasetDict
import pandas as pd

# Load from JSONL
train_df = pd.read_json("dataset_train.jsonl", lines=True)
val_df   = pd.read_json("dataset_val.jsonl",   lines=True)
test_df  = pd.read_json("dataset_test.jsonl",  lines=True)

dataset = DatasetDict({
    "train":      Dataset.from_pandas(train_df),
    "validation": Dataset.from_pandas(val_df),
    "test":       Dataset.from_pandas(test_df),
})

print(dataset)
# Push to Hub
dataset.push_to_hub("username/my-arabic-dataset", private=False)
```
