# 🧭 CDM Compass
**Clinical Data Management Knowledge Navigator**

Ask questions about your CDM/CDS documents in plain English and get answers with inline citations.
Documents are classified by folder. Text is chunked by paragraph so every chunk is a complete semantic unit — a full regulatory clause, a complete recommendation, a coherent argument. The interactive loop remembers your last few questions within a session.

**Example:**
> **You ask:** *"What does guidance say about RBQM?"*
> **Follow-up:** *"Then how does that apply to data cleaning?"*
>
> **Sources cited:**
> `⚖️ REGULATORY  🟢 HIGH  [1]  ICH-E6-R2.pdf  --  page 9`
> `💬 OPINION     🟡 MEDIUM  [2]  SCDM-Position-Paper-V9.pdf  --  page 8`

---

## ⚙️ One-Time Setup

### Step 1 — Install Ollama

1. Go to **[https://ollama.com](https://ollama.com)** and download the app
2. Open **Terminal** and run:
   ```bash
   ollama pull qwen2.5:14b
   ```
   (~9 GB download, once only)

### Step 2 — Create the document folders

```bash
mkdir -p data/raw/regulatory
mkdir -p data/raw/opinion
```

### Step 3 — Set up your Python environment

```bash
python3 -m venv .venv
source .venv/bin/activate          # on Windows: .venv\Scripts\activate
pip install "sentence-transformers[cross-encoder]" chromadb pypdf openpyxl python-docx python-pptx rank-bm25 requests tqdm jupyter ipywidgets
```

> Note the quotes around `sentence-transformers[cross-encoder]` — required in zsh (Mac default shell).

### Step 4 — Open the notebook

```bash
jupyter notebook CDM_Compass.ipynb
```

Run each block **from top to bottom**, in order.

---

## 📁 Adding Documents

### Where to put files — authority by folder

| Folder | Authority | Use for | LLM language |
|--------|-----------|---------|--------------|
| `data/raw/regulatory/` | ⚖️ Regulatory | ICH, FDA, EMA, binding standards | *must, is required, mandates, shall* |
| `data/raw/opinion/` | 💬 Opinion | SCDM, JSCDM, white papers, position papers | *recommends, suggests, proposes, advises* |

> ⚠️ Files placed anywhere else will raise an error — nothing is ever classified by accident.

### Supported file types

| Format | Extension | How it's read |
|--------|-----------|---------------|
| PDF | `.pdf` | Page by page |
| PowerPoint | `.pptx` | Slide by slide (incl. speaker notes) |
| Excel | `.xlsx`, `.xls` | Each sheet becomes one section |
| Word | `.docx` | Paragraph by paragraph |
| JSON | `.json` | Converted to readable text |
| XML | `.xml` | Element text extracted |
| CSV | `.csv` | Rows joined as text |
| Plain text | `.txt` | Read as-is |

> ⚠️ Old-format `.ppt` files are not supported. Open in PowerPoint → File → Save As → .pptx first.

### How to add a new document

1. Drop the file into the correct subfolder
2. Re-run **Blocks 3, 4, and 5**
3. Ask your question in Block 7 or 8

> 💡 You do **not** need to re-run Blocks 1, 2, or 6 unless you restart Jupyter.

### Managing document versions

| Situation | What to do |
|-----------|------------|
| Updated document, old version no longer valid | Delete old file, drop in new one with same filename, re-run Blocks 3–5 |
| Keep both versions for comparison | Rename with version numbers (e.g. `SCDM-v8.pdf` and `SCDM-v9.pdf`), keep both in same subfolder, re-run Blocks 3–5 |

---

## 🔍 How CDM Compass Finds Answers

Each question goes through a five-stage pipeline:

**Stage 1 — Paragraph chunking (Block 4)**
Text is split at natural paragraph boundaries rather than by a fixed word count. Each chunk is a complete semantic unit — a full regulatory clause, a complete recommendation — rather than an arbitrary word slice. Short paragraphs (below `MIN_PARA_WORDS=30`) are merged with the next; long ones (above `MAX_PARA_WORDS=400`) are split at sentence boundaries. A one-sentence overlap is added at each boundary to preserve context.

This means the embedding for each chunk captures one complete idea, the reranker can judge a coherent piece of text, and the LLM receives complete arguments rather than mid-sentence fragments.

**Stage 2 — Hybrid search**

| Index | Type | Good at |
|-------|------|---------|
| ChromaDB | Semantic (vector) | Meaning, synonyms, paraphrases |
| BM25 | Keyword | Exact terms, IDs, regulation numbers |

Default weighting: 70% semantic / 30% keyword (tunable in Block 7).

**Stage 3 — Reranking**
A cross-encoder (`ms-marco-MiniLM-L-12-v2`) re-scores all candidates by reading question + chunk together. More accurate than embedding similarity alone. Best `FINAL_K` chunks selected.

**Stage 4 — Contextual compression**
Each chunk is further compressed to its most relevant sentences. Reduces noise and improves answer focus.

**Stage 5 — Context-aware answer**
The last `HISTORY_TURNS` (default: 3) Q&A pairs are included as context. Follow-up questions work naturally. Set `HISTORY_TURNS = 0` in Block 7 to turn this off.

---

## 💬 Reading the Output

A token estimate is printed before each answer. If it exceeds `TOKEN_WARN_LIMIT` (default: 3000), reduce `HISTORY_TURNS` or `FINAL_K`.

### Answer
- Regulatory sources → *must, shall, is required*
- Opinion sources → *recommends, suggests, proposes*

Citations as `[1]`, `[2]` — bracket number only. Number in answer matches number in sources list.

### Sources cited
Sources appear **in the order they are cited in the answer**. Gaps (e.g. [1],[2],[4]) mean those chunks were retrieved but not cited by the LLM.

**Authority badge:**

| Badge | Meaning |
|-------|---------|
| ⚖️ REGULATORY | Binding requirement — ICH, FDA, EMA |
| 💬 OPINION | Professional recommendation — SCDM, JSCDM, white papers |

**Relevance badge** (hybrid score — lower = better):

| Badge | Score | Meaning |
|-------|-------|---------|
| 🟢 HIGH | < 0.5 | Strong match |
| 🟡 MEDIUM | 0.5 – 0.65 | Partial match |
| 🔴 LOW | > 0.65 | Weak match |

---

## 🔧 Tuning

**Chunking settings (Block 4) — re-run Blocks 4 and 5 after changing:**

| Setting | Default | Effect |
|---------|---------|--------|
| `MIN_PARA_WORDS` | 30 | Paragraphs shorter than this are merged with the next |
| `MAX_PARA_WORDS` | 400 | Paragraphs longer than this are split at sentence boundaries |

**Search and retrieval settings (Block 7):**

| Setting | Default | Effect |
|---------|---------|--------|
| `TOP_K` | 10 | Candidates before reranking — raise if answers miss known sources |
| `FINAL_K` | 5 | Chunks sent to LLM — raise for broader answers |
| `HISTORY_TURNS` | 3 | Conversation memory window (0 = off) |
| `TOKEN_WARN_LIMIT` | 3000 | Warn if estimated prompt size exceeds this |
| `SEMANTIC_WEIGHT` | 0.7 | Raise for conceptual questions |
| `KEYWORD_WEIGHT` | 0.3 | Raise for specific terms, IDs, clause numbers |

---

## 🔧 Changing the AI Model

Edit **Block 6**:

```python
OLLAMA_MODEL = "qwen2.5:14b"   # recommended
OLLAMA_MODEL = "mistral"       # good alternative
OLLAMA_MODEL = "llama3.2"      # also solid
OLLAMA_MODEL = "phi3"          # fastest — least reliable on citations
```

---

## ❓ Troubleshooting

| Symptom | Fix |
|---------|-----|
| *"Cannot connect to Ollama"* | Open the Ollama app, or run `ollama serve` in Terminal |
| *"Cannot classify: filename.pdf"* | Move the file into `regulatory/` or `opinion/` subfolder |
| *"Skipped (old format): file.ppt"* | Convert: PowerPoint → File → Save As → PowerPoint Presentation (.pptx) |
| `zsh: no matches found: sentence-transformers[...]` | Add quotes: `pip install "sentence-transformers[cross-encoder]"` |
| No sources listed after answer | LLM did not cite inline — try `qwen2.5:14b`, increase `TOP_K`, or rephrase |
| Token warning printed | Reduce `HISTORY_TURNS` or `FINAL_K` in Block 7 |
| Very few chunks created | Lower `MIN_PARA_WORDS` — your documents may have short paragraphs |
| Chunks seem too large | Lower `MAX_PARA_WORDS` from 400 to 250 |
| Block 5 slow on first run | Models download once and are cached after that |

---

## 🗂️ Project Structure

```
CDM-Compass/
├── CDM_Compass.ipynb          ← main notebook
├── README.md                  ← this file
├── claude.md                  ← repository knowledge for agents
├── agents.md                  ← agent patterns and issue log
├── done-tasks.md              ← completed task log
└── data/
    └── raw/
        ├── regulatory/        ← binding documents (ICH, FDA, EMA)
        └── opinion/           ← position papers, white papers
```

---

## ✅ Current Status

| Feature | Status |
|---------|--------|
| Load PDF, PPTX, XLSX, DOCX, JSON, XML, CSV, TXT | ✅ |
| Folder-based authority classification (regulatory / opinion) | ✅ |
| Error on unclassified files | ✅ |
| **Paragraph-based chunking** (complete semantic units) | ✅ |
| Hybrid search (semantic + BM25, tunable weights) | ✅ |
| Cross-encoder reranking (`ms-marco-MiniLM-L-12-v2`) | ✅ |
| Contextual compression (relevant sentences only) | ✅ |
| Conversation memory (rolling window, configurable) | ✅ |
| Token budget awareness (estimate + warning) | ✅ |
| Batched ChromaDB indexing | ✅ |
| Regulatory sources ranked first | ✅ |
| Authority-aware LLM language (must vs recommends) | ✅ |
| Citations as [N] only — no filename inline | ✅ |
| Sources in answer-citation order | ✅ |
| Only cited sources shown | ✅ |
| Colour-coded authority and relevance badges | ✅ |
| Wrapped, readable output | ✅ |
| Local LLM via Ollama (`qwen2.5:14b` recommended) | ✅ |
| Interactive Q&A loop with session memory (Block 8) | ✅ |

---

## 🔮 Future Ideas

- **Fine-tuning the reranker on CDM data** — training on CDM-specific Q&A pairs would dramatically improve relevance for specialist terminology (same domain-vocabulary problem as legal documents, where accuracy went 30% → 95% on just 72 training pairs)
- **Persistent memory across sessions** — save conversation history to SQLite so context survives Jupyter restarts
- **Hybrid search weight auto-tuning** — detect query type and adjust weights automatically
- **Agentic retrieval** — let the LLM re-query if first results are insufficient
- **Semantic caching** — reuse answers for similar questions (useful if CDM Compass becomes a shared team tool)
- **CLI tool** for faster queries without Jupyter
- **Docker packaging** for reproducible deployment

> 💡 Documents are not included in this repository. Place your own files in `data/raw/regulatory/` or `data/raw/opinion/`.
