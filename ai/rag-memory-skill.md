# RAG & Memory Skills — Retrieval Augmented Generation

## RAG Pipeline Overview
```
Document → Chunk → Embed → Store (VectorDB)
                                    ↓
Query → Embed → Similarity Search → Top-K Chunks → LLM Prompt → Answer
```

## Full RAG Implementation
```python
from langchain_community.document_loaders import PyPDFLoader, DirectoryLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.embeddings import HuggingFaceEmbeddings
from langchain_community.vectorstores import Chroma
from langchain.chains import RetrievalQA
from langchain_anthropic import ChatAnthropic

# 1. Load documents
loader = DirectoryLoader("./docs", glob="**/*.pdf", loader_cls=PyPDFLoader)
docs   = loader.load()
print(f"Loaded {len(docs)} pages")

# 2. Chunk
splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000, chunk_overlap=200,
    separators=["\n\n", "\n", ".", " "])
chunks = splitter.split_documents(docs)

# 3. Embed + Store
embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/paraphrase-multilingual-mpnet-base-v2",
    model_kwargs={"device": "cpu"})
vectordb = Chroma.from_documents(chunks, embeddings,
                                  persist_directory="./chroma_db")
vectordb.persist()

# 4. Query
retriever = vectordb.as_retriever(search_type="mmr",
                                   search_kwargs={"k": 5, "fetch_k": 20})
llm       = ChatAnthropic(model="claude-sonnet-4-20250514")
chain     = RetrievalQA.from_chain_type(llm=llm, retriever=retriever,
                                         return_source_documents=True)

result = chain({"query": "ما هي شروط التسجيل في الجامعة؟"})
print(result["result"])
for doc in result["source_documents"]:
    print(f"  Source: {doc.metadata['source']} p.{doc.metadata.get('page','?')}")
```

## Custom Chunking Strategies
```python
def semantic_chunk(text: str, max_size: int = 800) -> list[str]:
    """Split by paragraphs, never mid-sentence."""
    paragraphs = [p.strip() for p in text.split("\n\n") if p.strip()]
    chunks, current = [], ""
    for para in paragraphs:
        if len(current) + len(para) < max_size:
            current += "\n\n" + para if current else para
        else:
            if current: chunks.append(current)
            current = para
    if current: chunks.append(current)
    return chunks

def sliding_window_chunk(text: str, size=500, overlap=100) -> list[str]:
    words  = text.split()
    chunks = []
    step   = size - overlap
    for i in range(0, len(words), step):
        chunk = " ".join(words[i:i+size])
        if chunk: chunks.append(chunk)
    return chunks
```

## Hybrid Search (Dense + Sparse)
```python
from rank_bm25 import BM25Okapi
import numpy as np

class HybridRetriever:
    def __init__(self, docs: list[str], embedder, alpha=0.5):
        self.docs     = docs
        self.embedder = embedder
        self.alpha    = alpha  # 0=BM25 only, 1=dense only
        # Sparse index
        tokenized = [d.lower().split() for d in docs]
        self.bm25 = BM25Okapi(tokenized)
        # Dense index
        self.vecs = np.array(embedder.encode(docs))

    def search(self, query: str, k=5) -> list[tuple[str, float]]:
        # BM25 scores
        bm25_scores = np.array(self.bm25.get_scores(query.lower().split()))
        bm25_scores = (bm25_scores - bm25_scores.min()) / (bm25_scores.max() - bm25_scores.min() + 1e-9)

        # Dense scores
        q_vec       = self.embedder.encode([query])
        dense_scores = (self.vecs @ q_vec.T).flatten()
        dense_scores = (dense_scores - dense_scores.min()) / (dense_scores.max() - dense_scores.min() + 1e-9)

        # Combine
        scores = (1-self.alpha)*bm25_scores + self.alpha*dense_scores
        top_k  = np.argsort(scores)[::-1][:k]
        return [(self.docs[i], float(scores[i])) for i in top_k]
```

## Agent Memory System
```python
import json, time
from pathlib import Path
from sentence_transformers import SentenceTransformer
import numpy as np

class AgentMemory:
    def __init__(self, persist_file="memory.json", embed_model="all-MiniLM-L6-v2"):
        self.file    = Path(persist_file)
        self.encoder = SentenceTransformer(embed_model)
        self.memories: list[dict] = []
        self.embeddings: list[np.ndarray] = []
        if self.file.exists():
            self._load()

    def remember(self, content: str, tags: list[str] = None):
        emb = self.encoder.encode(content)
        entry = {"id": len(self.memories), "content": content,
                 "tags": tags or [], "ts": time.time()}
        self.memories.append(entry)
        self.embeddings.append(emb)
        self._save()

    def recall(self, query: str, top_k=5, min_score=0.4) -> list[dict]:
        if not self.memories: return []
        q_emb = self.encoder.encode(query)
        scores = np.array(self.embeddings) @ q_emb / (
            np.linalg.norm(self.embeddings, axis=1) * np.linalg.norm(q_emb) + 1e-9)
        idxs = np.argsort(scores)[::-1][:top_k]
        return [self.memories[i] | {"score": float(scores[i])}
                for i in idxs if scores[i] >= min_score]

    def forget(self, tag: str):
        keep_idx = [i for i,m in enumerate(self.memories) if tag not in m["tags"]]
        self.memories   = [self.memories[i] for i in keep_idx]
        self.embeddings = [self.embeddings[i] for i in keep_idx]
        self._save()

    def _save(self):
        self.file.write_text(json.dumps({"memories": self.memories}))

    def _load(self):
        data = json.loads(self.file.read_text())
        self.memories = data.get("memories", [])
        if self.memories:
            self.embeddings = [self.encoder.encode(m["content"]) for m in self.memories]

    def context_for_prompt(self, query: str, max_items=5) -> str:
        results = self.recall(query, top_k=max_items)
        if not results: return ""
        return "Relevant context from memory:\n" + \
               "\n".join(f"- {r['content']}" for r in results)
```

## RAG with Re-ranking
```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rag_with_reranking(query: str, retriever, llm, top_k=10, final_k=3):
    # Step 1: retrieve more than needed
    initial_docs = retriever.get_relevant_documents(query)[:top_k]

    # Step 2: re-rank with cross-encoder (much more accurate)
    pairs  = [[query, doc.page_content] for doc in initial_docs]
    scores = reranker.predict(pairs)
    ranked = sorted(zip(initial_docs, scores), key=lambda x: x[1], reverse=True)
    top_docs = [doc for doc, _ in ranked[:final_k]]

    # Step 3: generate with best docs
    context = "\n\n---\n\n".join(d.page_content for d in top_docs)
    prompt  = f"Context:\n{context}\n\nQuestion: {query}\nAnswer:"
    return llm.invoke(prompt).content, top_docs
```

## Editing Existing Content with RAG
```python
def rag_edit(document: str, instruction: str, rag_context: str, llm) -> str:
    """Edit any document using RAG context + LLM."""
    prompt = f"""You are an expert editor. Edit the document below following the instruction.
Use the reference context if relevant.

INSTRUCTION: {instruction}

REFERENCE CONTEXT:
{rag_context}

DOCUMENT TO EDIT:
{document}

EDITED DOCUMENT:"""
    return llm.invoke(prompt).content
```
