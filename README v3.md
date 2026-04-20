# 🧭 CDM Compass
**Clinical Data Management Knowledge Navigator**

Ask questions about your CDM/CDS documents in plain English and get answers with inline citations.
Documents are classified by the folder they live in. The interactive loop remembers your last few questions within a session, so follow-up questions work naturally.

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

Each question goes through a four-stage pipeline:

**Stage 1 — Hybrid search**

| Index | Type | Good at |
|-------|------|---------|
| ChromaDB | Semantic (vector) | Meaning, synonyms, paraphrases |
| BM25 | Keyword | Exact terms, IDs, regulation numbers |

Default weighting: 70% semantic / 30% keyword (tunable in Block 7).

**Stage 2 — Reranking**
A cross-encoder (`ms-marco-MiniLM-L-12-v2`) re-scores all candidates by reading question + chunk together. More accurate than embedding similarity alone. Best `FINAL_K` chunks selected.

**Stage 3 — Contextual compression**
Each chunk is compressed to its most relevant sentences before reaching the LLM. Reduces noise, improves answer focus.

**Stage 4 — Context-aware answer**
The last `HISTORY_TURNS` (default: 3) question/answer pairs from the session are included as context. Follow-up questions like *"Then how does that apply to RBQM?"* work naturally. Set `HISTORY_TURNS = 0` in Block 7 to turn this off.

---

## 💬 Reading the Output

A token estimate is printed before each answer. If it exceeds `TOKEN_WARN_LIMIT` (default: 3000 tokens), CDM Compass will warn you — reduce `HISTORY_TURNS` or `FINAL_K` if this happens regularly.

### Answer
Language is calibrated to source authority automatically:
- Regulatory → *must, shall, is required*
- Opinion → *recommends, suggests, proposes*

Citations appear as `[1]`, `[2]` etc. — bracket number only. The number in the answer matches the number in the sources list.

### Sources cited
Sources appear **in the order they are cited in the answer**. Gaps (e.g. [1],[2],[4]) mean those chunks were retrieved but not used by the LLM.

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

Each source also shows its rerank score (higher = better), semantic score, and keyword score.

---

## 🔧 Tuning

All settings are at the top of Block 7:

| Setting | Default | Effect |
|---------|---------|--------|
| `TOP_K` | 10 | Candidates before reranking — raise if answers miss known sources |
| `FINAL_K` | 5 | Chunks sent to LLM — raise for broader answers |
| `HISTORY_TURNS` | 3 | Conversation memory window (0 = off) |
| `TOKEN_WARN_LIMIT` | 3000 | Warn if estimated prompt size exceeds this |
| `SEMANTIC_WEIGHT` | 0.7 | Raise for conceptual questions |
| `KEYWORD_WEIGHT` | 0.3 | Raise for specific terms, IDs, clause numbers |
| `CHUNK_SIZE` (Block 4) | 300 | Lower (e.g. 200) for more precise citations |

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
| Follow-up questions not working | Make sure `HISTORY_TURNS > 0` in Block 7 and you are using Block 8 |
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

- **Fine-tuning the reranker on CDM data** — training `ms-marco-MiniLM-L-12-v2` on CDM-specific Q&A pairs would dramatically improve relevance for specialist terms (same domain-vocabulary problem demonstrated in legal document fine-tuning, where accuracy went from 30% to 95% on just 72 training pairs)
- **Persistent memory across sessions** — save conversation history to SQLite so context survives Jupyter restarts
- **Hybrid search weight auto-tuning** — detect query type and adjust weights automatically
- **Agentic retrieval** — let the LLM re-query if first results are insufficient
- **Semantic caching** — reuse answers for similar questions (useful if CDM Compass becomes a shared team tool)
- **Improved heading extraction** from PDFs
- **CLI tool** for faster queries without Jupyter
- **Docker packaging** for reproducible deployment

> 💡 Documents are not included in this repository. Place your own files in `data/raw/regulatory/` or `data/raw/opinion/`.
