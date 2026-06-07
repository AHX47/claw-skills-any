# AI Model Training Skills

## Training Pipeline Overview
```
Data Collection → Preprocessing → Model Design → Training Loop → Evaluation → Deployment
```

## Dataset Preparation
```python
import pandas as pd
from sklearn.model_selection import train_test_split

df = pd.read_csv("data.csv")

# Split: 70% train, 15% val, 15% test
train, temp = train_test_split(df, test_size=0.30, random_state=42, stratify=df["label"])
val, test   = train_test_split(temp, test_size=0.50, random_state=42)

print(f"Train: {len(train)} | Val: {len(val)} | Test: {len(test)}")
print(f"Label distribution:\n{df['label'].value_counts(normalize=True)}")
```

## PyTorch Training Loop
```python
import torch, torch.nn as nn
from torch.utils.data import DataLoader, Dataset

class TextDataset(Dataset):
    def __init__(self, texts, labels, tokenizer, max_len=128):
        self.encodings = tokenizer(texts, truncation=True,
                                   padding=True, max_length=max_len,
                                   return_tensors="pt")
        self.labels = torch.tensor(labels)

    def __len__(self): return len(self.labels)
    def __getitem__(self, i):
        return {k: v[i] for k, v in self.encodings.items()}, self.labels[i]


def train_epoch(model, loader, optimizer, criterion, device, scaler=None):
    model.train()
    total_loss, correct, total = 0, 0, 0
    for batch_inputs, labels in loader:
        batch_inputs = {k: v.to(device) for k, v in batch_inputs.items()}
        labels = labels.to(device)
        optimizer.zero_grad()
        if scaler:  # mixed precision
            with torch.cuda.amp.autocast():
                logits = model(**batch_inputs).logits
                loss   = criterion(logits, labels)
            scaler.scale(loss).backward()
            scaler.unscale_(optimizer)
            torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
            scaler.step(optimizer); scaler.update()
        else:
            logits = model(**batch_inputs).logits
            loss   = criterion(logits, labels)
            loss.backward()
            torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
            optimizer.step()
        total_loss += loss.item()
        correct    += (logits.argmax(1) == labels).sum().item()
        total      += labels.size(0)
    return total_loss / len(loader), correct / total


def evaluate(model, loader, criterion, device):
    model.eval()
    total_loss, correct, total = 0, 0, 0
    with torch.no_grad():
        for batch_inputs, labels in loader:
            batch_inputs = {k: v.to(device) for k, v in batch_inputs.items()}
            labels = labels.to(device)
            logits = model(**batch_inputs).logits
            loss   = criterion(logits, labels)
            total_loss += loss.item()
            correct    += (logits.argmax(1) == labels).sum().item()
            total      += labels.size(0)
    return total_loss / len(loader), correct / total
```

## Fine-tuning with HuggingFace
```python
from transformers import (AutoTokenizer, AutoModelForSequenceClassification,
                           TrainingArguments, Trainer)
from datasets import load_dataset
import evaluate, numpy as np

model_name = "bert-base-multilingual-cased"  # or "CAMeL-Lab/bert-base-arabic-camelbert-mix"
tokenizer  = AutoTokenizer.from_pretrained(model_name)
model      = AutoModelForSequenceClassification.from_pretrained(model_name, num_labels=2)

dataset    = load_dataset("csv", data_files={"train":"train.csv","validation":"val.csv"})

def tokenize(examples):
    return tokenizer(examples["text"], truncation=True, padding="max_length", max_length=128)

tokenized = dataset.map(tokenize, batched=True)
accuracy  = evaluate.load("accuracy")

def compute_metrics(eval_pred):
    logits, labels = eval_pred
    preds = np.argmax(logits, axis=-1)
    return accuracy.compute(predictions=preds, references=labels)

args = TrainingArguments(
    output_dir="./results",
    num_train_epochs=3,
    per_device_train_batch_size=16,
    per_device_eval_batch_size=32,
    warmup_ratio=0.1,
    weight_decay=0.01,
    learning_rate=2e-5,
    evaluation_strategy="epoch",
    save_strategy="epoch",
    load_best_model_at_end=True,
    metric_for_best_model="accuracy",
    fp16=True,
    report_to="none",
)

trainer = Trainer(model=model, args=args,
                   train_dataset=tokenized["train"],
                   eval_dataset=tokenized["validation"],
                   compute_metrics=compute_metrics)
trainer.train()
trainer.save_model("./best_model")
```

## LoRA / QLoRA Fine-tuning (LLMs)
```python
from peft import LoraConfig, get_peft_model, TaskType
from transformers import AutoModelForCausalLM, BitsAndBytesConfig
import torch

# QLoRA: 4-bit quantized + LoRA
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_use_double_quant=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.float16,
)
model = AutoModelForCausalLM.from_pretrained(
    "mistralai/Mistral-7B-v0.1",
    quantization_config=bnb_config,
    device_map="auto",
)
model = get_peft_model(model, LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=16, lora_alpha=32, lora_dropout=0.05,
    target_modules=["q_proj","v_proj","k_proj","o_proj"],
))
model.print_trainable_parameters()
# trainable params: 6.8M || all params: 3.75B || trainable%: 0.18%
```

## GGUF Export (for local inference)
```bash
# Convert HuggingFace model to GGUF
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp && pip install -r requirements.txt

python convert_hf_to_gguf.py ./my_model \
  --outfile ./my_model.gguf \
  --outtype q4_k_m   # quantization level

# Run locally
./llama-cli -m my_model.gguf -p "مرحباً، كيف يمكنني مساعدتك؟"
```

## Jupyter Notebook Training (ipynb)
```python
# Cell 1 — Setup
import os; os.environ["TOKENIZERS_PARALLELISM"] = "false"
!pip install transformers datasets accelerate peft -q

# Cell 2 — Config
CONFIG = {
    "model":      "aubmindlab/bert-base-arabertv02",
    "max_len":    128,
    "batch_size": 32,
    "epochs":     5,
    "lr":         3e-5,
    "output":     "./arabic_classifier",
}

# Cell 3 — Verify GPU
import torch
print(f"GPU: {torch.cuda.get_device_name(0)}")
print(f"VRAM: {torch.cuda.get_device_properties(0).total_memory/1e9:.1f}GB")

# Cell 4 — Training...
```

## Evaluation Metrics
```python
from sklearn.metrics import classification_report, confusion_matrix
import seaborn as sns, matplotlib.pyplot as plt

def full_evaluation(y_true, y_pred, class_names):
    print(classification_report(y_true, y_pred, target_names=class_names))
    cm = confusion_matrix(y_true, y_pred)
    plt.figure(figsize=(8,6))
    sns.heatmap(cm, annot=True, fmt="d", xticklabels=class_names, yticklabels=class_names)
    plt.title("Confusion Matrix"); plt.show()
```

## Experiment Tracking
```python
import mlflow

with mlflow.start_run(run_name="bert-arabic-v1"):
    mlflow.log_params(CONFIG)
    for epoch in range(CONFIG["epochs"]):
        train_loss, train_acc = train_epoch(...)
        val_loss,   val_acc   = evaluate(...)
        mlflow.log_metrics({"train_loss": train_loss, "val_acc": val_acc}, step=epoch)
    mlflow.pytorch.log_model(model, "model")
```
