# 🧭 CDM Compass
**Clinical Data Management Knowledge Navigator**

Ask questions about your CDM/CDS documents in plain English and get answers with inline citations.
Documents are classified by folder. Text is chunked according to document type — structured documents
(numbered guidelines, specifications) get hierarchical chunking that preserves section context; narrative
documents (position papers, prose) get paragraph chunking; tables are converted to readable sentences
so their data is retrievable rather than silently lost.

**Example:**
> **You ask:** *"What does guidance say about RBQM?"*
> **Follow-up:** *"Then how does that apply to data cleaning?"*
>
> **Sources cited:**
> `⚖️ REGULATORY  🟢 HIGH  [1]  ICH-E6-R2.pdf  --  page 9  [hier_leaf]`
> `💬 OPINION     🟡 MEDIUM  [2]  SCDM-Position-Paper-V9.pdf  --  page 8  [paragraph]`

---

## ⚙️ One-Time Setup

### Step 1 — Install Ollama

1. Go to **[https://ollama.com](https://ollama.com)** and download the app
2. Open **Terminal** and run:
   ```bash
   ollama pull qwen2.5:14b
   ```

### Step 2 — Create the document folders

```bash
mkdir -p data/raw/regulatory
mkdir -p data/raw/opinion
```

### Step 3 — Set up your Python environment

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install "sentence-transformers[cross-encoder]" chromadb pypdf pdfplumber openpyxl python-docx python-pptx rank-bm25 requests tqdm jupyter ipywidgets
```

> Note the quotes around `sentence-transformers[cross-encoder]` — required in zsh (Mac default shell).

### Step 4 — Open the notebook

```bash
jupyter notebook CDM_Compass.ipynb
```

---

## 📁 Adding Documents

### Where to put files — authority by folder

| Folder | Authority | Use for | LLM language |
|--------|-----------|---------|--------------|
| `data/raw/regulatory/` | ⚖️ Regulatory | ICH, FDA, EMA, binding standards | *must, is required, mandates, shall* |
| `data/raw/opinion/` | 💬 Opinion | SCDM, JSCDM, white papers, position papers | *recommends, suggests, proposes, advises* |

> ⚠️ Files placed anywhere else will raise an error.

### Supported file types

| Format | Extension | How it's read |
|--------|-----------|---------------|
| PDF | `.pdf` | Page by page (prose) + table rows extracted separately |
| PowerPoint | `.pptx` | Slide by slide (incl. speaker notes) |
| Excel | `.xlsx`, `.xls` | Each row converted to a readable sentence |
| Word | `.docx` | Full document, doc type auto-detected |
| JSON | `.json` | Converted to readable text |
| XML | `.xml` | Element text extracted |
| CSV | `.csv` | Each row converted to a readable sentence |
| Plain text | `.txt` | Full file, doc type auto-detected |

> ⚠️ Old-format `.ppt` files are not supported. Open in PowerPoint → File → Save As → .pptx first.

### How to add a new document

1. Drop the file into the correct subfolder
2. Re-run **Blocks 3, 4, and 5**
3. Ask your question in Block 7 or 8

### Managing document versions

| Situation | What to do |
|-----------|------------|
| Updated document, old version no longer valid | Delete old file, drop in new one with same filename, re-run Blocks 3–5 |
| Keep both versions | Rename with version numbers, keep in same subfolder, re-run Blocks 3–5 |

---

## 🔍 How CDM Compass Finds Answers

### Stage 1 — Type-aware chunking

CDM Compass detects the structure of each document and routes it to the right chunking strategy:

| Document type | Detection | Strategy | Why |
|---------------|-----------|----------|-----|
| `structured` | >8% of lines look like headings or numbered sections | **Hierarchical** | Preserves section context alongside paragraph precision |
| `narrative` | Flowing prose, low heading density | **Paragraph** | Complete semantic units — full arguments, full clauses |
| `table` | XLSX, CSV, PDF tables | **Row sentences** | Each row → `"Header1: value1, Header2: value2."` |

**Why tables need special treatment:** A flattened table like `"Product A EMEA 4.2M 12%"` is meaningless to an embedding model. Converting it to `"Product: Product A, Region: EMEA, Q3 Revenue: 4.2M, YoY Growth: 12%."` makes it both retrievable and interpretable.

**Why hierarchical chunking helps structured documents:** ICH guidelines and numbered specifications have meaningful section structure. Hierarchical chunking stores both section-level chunks (for broad queries that touch multiple sub-points) and paragraph-level leaf chunks (for precise queries). If a query matches multiple leaves from the same section, the section context is available.

The chunk type (`hier_section`, `hier_leaf`, `paragraph`, `table`) is shown in the sources list so you can see which strategy produced each cited result.

### Stage 2 — Hybrid search
ChromaDB (semantic, 70%) + BM25 (keyword, 30%), tunable in Block 7.

### Stage 3 — Reranking
`ms-marco-MiniLM-L-12-v2` cross-encoder re-scores all candidates. Best `FINAL_K` selected.

### Stage 4 — Contextual compression
Each chunk compressed to most relevant sentences before reaching the LLM.

### Stage 5 — Context-aware answer
Last `HISTORY_TURNS` (default: 3) Q&A pairs included so follow-up questions work.

---

## 💬 Reading the Output

A token estimate is printed before each answer. If it exceeds `TOKEN_WARN_LIMIT` (3000), reduce `HISTORY_TURNS` or `FINAL_K`.

### Answer
- Regulatory sources → *must, shall, is required*
- Opinion sources → *recommends, suggests, proposes*

Citations as `[1]`, `[2]` — bracket number only.

### Sources cited
In the order cited. Gaps (e.g. [1],[2],[4]) = retrieved but not cited. Each source shows its chunk type.

| Authority badge | Meaning |
|-----------------|---------|
| ⚖️ REGULATORY | Binding — ICH, FDA, EMA |
| 💬 OPINION | Recommendation — SCDM, JSCDM, white papers |

| Relevance badge | Hybrid score | Meaning |
|----------------|--------------|---------|
| 🟢 HIGH | < 0.5 | Strong match |
| 🟡 MEDIUM | 0.5 – 0.65 | Partial match |
| 🔴 LOW | > 0.65 | Weak match |

| Chunk type | Meaning |
|------------|---------|
| `hier_section` | Full section from a structured document |
| `hier_leaf` | Individual paragraph within a section |
| `paragraph` | Paragraph from a narrative document |
| `table` | Table row as a readable sentence |

---

## 🔧 Tuning

**Chunking (Block 4) — re-run Blocks 4 and 5 after changing:**

| Setting | Default | Effect |
|---------|---------|--------|
| `MIN_PARA_WORDS` | 30 | Merge paragraphs shorter than this with the next |
| `MAX_PARA_WORDS` | 400 | Split paragraphs longer than this at sentence boundaries |
| `HIER_SECTION_WORDS` | 800 | Max words in a section-level chunk |

**Search (Block 7):**

| Setting | Default | Effect |
|---------|---------|--------|
| `TOP_K` | 10 | Candidates before reranking |
| `FINAL_K` | 5 | Chunks sent to LLM |
| `HISTORY_TURNS` | 3 | Conversation memory (0 = off) |
| `TOKEN_WARN_LIMIT` | 3000 | Warn if prompt exceeds this |
| `SEMANTIC_WEIGHT` | 0.7 | Raise for conceptual questions |
| `KEYWORD_WEIGHT` | 0.3 | Raise for exact terms, IDs, clause numbers |

---

## ❓ Troubleshooting

| Symptom | Fix |
|---------|-----|
| *"Cannot connect to Ollama"* | Open Ollama app, or run `ollama serve` in Terminal |
| *"Cannot classify: filename.pdf"* | Move file into `regulatory/` or `opinion/` subfolder |
| *"Skipped (old format): file.ppt"* | Convert: PowerPoint → File → Save As → .pptx |
| `zsh: no matches found` on pip install | Use quotes: `pip install "sentence-transformers[cross-encoder]"` |
| No sources listed | LLM did not cite inline — try `qwen2.5:14b`, increase `TOP_K`, rephrase |
| Token warning | Reduce `HISTORY_TURNS` or `FINAL_K` in Block 7 |
| Very few chunks | Lower `MIN_PARA_WORDS` — your docs may have many short paragraphs |
| Chunks seem too large | Lower `MAX_PARA_WORDS` from 400 to 250 |
| Block 5 slow on first run | Models download once and are cached after that |

---

## 🗂️ Project Structure

```
CDM-Compass/
├── CDM_Compass.ipynb
├── README.md
├── claude.md
├── agents.md
├── done-tasks.md
└── data/
    └── raw/
        ├── regulatory/
        └── opinion/
```

---

## ✅ Current Status

| Feature | Status |
|---------|--------|
| Load PDF, PPTX, XLSX, DOCX, JSON, XML, CSV, TXT | ✅ |
| PDF table extraction via pdfplumber → readable sentences | ✅ |
| XLSX / CSV rows → readable header:value sentences | ✅ |
| Document type detection (structured vs narrative) | ✅ |
| Hierarchical chunking for structured documents | ✅ |
| Paragraph chunking for narrative documents | ✅ |
| Folder-based authority classification (regulatory / opinion) | ✅ |
| Hybrid search (semantic + BM25, tunable weights) | ✅ |
| Cross-encoder reranking (ms-marco-MiniLM-L-12-v2) | ✅ |
| Contextual compression | ✅ |
| Conversation memory (rolling window) | ✅ |
| Token budget awareness | ✅ |
| Chunk type shown in sources list | ✅ |
| Citations as [N] only, in answer-citation order | ✅ |
| Only cited sources shown | ✅ |
| Local LLM via Ollama (qwen2.5:14b recommended) | ✅ |
| Interactive Q&A with session memory | ✅ |

---

## 🔮 Future Ideas

- **RAGAS evaluation** — measure context recall, context precision, faithfulness, and answer relevancy before and after chunking changes; gives objective evidence of whether improvements actually help
- **Fine-tuning the reranker on CDM data** — BCE fine-tuning on CDM-specific Q&A pairs; accuracy went 30% → 95% on legal documents with just 72 training pairs
- **Persistent memory across sessions** — save conversation history to SQLite
- **Hybrid search weight auto-tuning** — classify query type and adjust weights automatically
- **Agentic retrieval** — let the LLM re-query if first results are insufficient
- **Semantic caching** — reuse answers for similar questions
- **CLI tool** / **Docker packaging**

> 💡 Documents are not included in this repository. Place your own files in `data/raw/regulatory/` or `data/raw/opinion/`.
