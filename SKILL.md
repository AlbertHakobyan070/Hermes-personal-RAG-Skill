---
name: personal-rag
title: Albert's Personal RAG (Vault Knowledge Retrieval)
description: "Retrieve grounded, cited answers from Albert's OWN AUA coursework, lecture notes, textbooks, and notebooks. Use when he asks what he knows about a topic, how he did something in his own homework/capstone/coursework, or wants to drill his own materials for interview/exam prep. NOT for general-knowledge questions unrelated to his vault."
version: 0.1.0
author: Albert Hakobyan
license: MIT
dependencies: [chromadb, sentence-transformers, bm25s, openai, pyyaml, python-dotenv, streamlit, pymupdf, pymupdf4llm]
platforms: [windows]
metadata:
  hermes:
    tags: [RAG, Retrieval, Notes, Obsidian, Study, Interview Prep, Citations, NLP, ML, Statistics, Time Series, Coursework, Recall]
    category: knowledge
    related_skills: [hermes-windows-setup]
    requires_toolsets: [terminal, files]
---

# Albert's Personal RAG — Vault Knowledge Retrieval

A hybrid-retrieval RAG over Albert's ~100GB Obsidian vault: 24+ AUA courses across
9 domains (NLP, ML, Statistics, Time Series, BI, Data Viz, Databases, Math,
Programming), plus 281 textbooks, lecture PDFs, passed coursework, and his own
notebooks/code. ~156K chunks. Answers are **grounded in Albert's own materials**
and carry inline `[n]` citations back to the source note/book/page.

**The core value:** when Albert asks "what do I know about X" or "how did I do Y
in my coursework," this retrieves from *his* notes — not the open internet. It is
a recall engine for his own knowledge, built to fight interview/exam imposter
syndrome and to surface his own prior code/technique during project work.

> This skill is a **playbook for invoking the RAG**, which lives as a Python
> project at `A:\DS_Vault\DS Main Vault\rag_project`. The agent drives it through
> the `terminal` toolset. It does not reimplement retrieval.

---

## When To Use This Skill

Trigger this skill (automatically, or when loaded via `/skill personal-rag`) when
Albert:
- Asks **"what do I know about X"**, "explain X from my notes", "how did I cover Y in <course>"
- Asks **"how did I implement / do Z in my homework / capstone / coursework"** (recall his own code or method)
- Is **prepping for an interview or exam** and wants to drill a topic against his own materials
- Is **building a project** and wants to check what technique/textbook code he already has for a sub-problem
- Wants a **grounded, cited** answer rather than a general-knowledge one — i.e. answers traceable to his vault

Do **NOT** use this skill for:
- General-knowledge questions with no tie to Albert's materials (answer directly or web-search instead) — e.g. "what's the weather", "who won the match", "write me an email"
- Questions about content the vault is known not to cover. If retrieval punts, report that honestly rather than filling from general knowledge.

When in doubt about whether a question is "about his vault," a quick tell: if the
question uses "I/my" about coursework, or names one of his courses/topics, it is a
RAG question. If it is a generic fact lookup, it is not.

---

## Core Philosophy

1. **Grounded over fluent.** The point is answers from Albert's own notes with
   citations. If the RAG says it has no relevant content, *report that* — do not
   silently substitute general knowledge. A truthful "your notes don't cover this"
   is more useful than a confident unsourced answer.
2. **Cite the source.** Always surface the `[n]` citations and their labels
   (course › heading, or book › chapter, p.N) so Albert can jump to the original.
3. **Prefer the warm path.** One-shot `query` reloads a ~350MB index every call
   (10–30s off the HDD). For more than one question in a session, use `chat`
   (terminal, persistent) or `serve` (browser app). See Latency below.
4. **Don't fabricate retrieval.** Never invent citations or claim the vault
   contains something it returned no chunks for. The whole skill is trust.

---

## Environment (Windows — REQUIRED, read before any command)

This project runs **only inside the Hermes venv**. A bare PowerShell/cmd lacks the
packages and every command will fail with ModuleNotFoundError.

**Project root:** `A:\DS_Vault\DS Main Vault\rag_project`
**Venv:** `C:\Users\Albert\AppData\Local\hermes\hermes-agent\venv` (Python 3.11)

Before running any command, activate the venv and cd to the project:

```powershell
C:\Users\Albert\AppData\Local\hermes\hermes-agent\venv\Scripts\activate
cd "A:\DS_Vault\DS Main Vault\rag_project"
```

(If a `rag.bat` launcher exists in the project root, prefer it — it activates the
venv and forwards args: `rag query "..."`. As of v0.1 it may not exist yet.)

**Generation requires the FreeLLMAPI proxy running** at `http://localhost:3001`.
If queries fail to generate (connection error), the proxy is down — tell Albert
to start FreeLLMAPI; do not try to fix it from the terminal. Verify with:

```powershell
curl.exe http://localhost:3001/v1/models
```

Embeddings are local (bge-small, CPU) and never need the proxy.

---

## Commands

All commands are run from the project root inside the venv (see Environment).

### Ask a single question (cold, slow — one-off only)

```powershell
python main.py query "How did I use knowledge distillation in my capstone?"
```

Reloads the index each call (10–30s). Fine for a single isolated question.

### Interactive session (warm, fast — preferred for study/drilling)

```powershell
python main.py chat
```

Loads the pipeline once, then answers many questions in a terminal loop. Use this
whenever Albert will ask more than one question.

### Web app (warm, richest view — preferred for interview prep)

```powershell
python -m streamlit run app.py
```

Opens `localhost:8501` with answer + confidence badge + expandable cited sources.
NOTE: use `python -m streamlit`, NOT `streamlit` directly and NOT
`python main.py serve` — the bare `streamlit` executable is not on PATH and
`main.py serve` shells out to it (known to fail with WinError 2 until patched).

### Run the eval suite (benchmark)

```powershell
python main.py eval
```

Runs 30 golden questions across 9 domains; prints keyword recall, retrieval hit
rate, confidence distribution, speed. For a *reproducible* number, pin a model
first (`generation.model: "llama-3.3-70b-versatile"` in config.yaml), run, then
revert to `"auto"`. On `auto` the number wobbles run-to-run because the router
serves different models.

### Add new content to the index (incremental)

```powershell
# notebooks / code (.ipynb/.py/.R/.Rmd)
python main.py ingest-notebooks
python main.py index --append data\ipynb_chunks.jsonl

# PDFs (books / lectures / coursework — scoped passes)
python main.py ingest-pdfs --skip-books --include-path "Current Courses"
python main.py index --append data\lecture_chunks.jsonl

# after ANY ingest or metadata change, rebuild the sparse index:
python rebuild_bm25.py
```

Ingestion is idempotent (deterministic doc_id + ChromaDB upsert) — re-running the
same content does not duplicate it. The JSONL files in `data\` are the source of
truth; ChromaDB and the BM25 pickle are derived and rebuildable.

---

## How To Present Results To Albert

When you run a query, report:
1. **The answer text** (it already contains inline `[n]` markers).
2. **The confidence** the model reported (HIGH/MEDIUM/LOW/UNKNOWN).
3. **The cited sources** — each `[n]` with its label (course › heading, or
   book › chapter, p.N). This is what lets Albert verify against the original.

If the RAG returns "your notes don't contain anything relevant," report that
plainly. Optionally offer to answer from general knowledge *as a clearly separate,
non-grounded* answer — but never blend the two.

If confidence is LOW/UNKNOWN or citations look thin, say so. Do not oversell a
weak grounded answer.

---

## Latency & Gotchas (read before debugging)

- **One-shot `query` is slow by design** (index reload). Not a bug. Use `chat`/`serve`.
- **Proxy down** = generation fails. Embeddings/retrieval still work; the failure
  is the LLM call. Tell Albert to start FreeLLMAPI; don't fix from terminal.
- **Bare `streamlit` / `main.py serve` fails** (WinError 2). Use `python -m streamlit run app.py`.
- **A Hermes app reinstall can wipe venv packages** (it has before — chromadb,
  sentence_transformers). If a command dies with ModuleNotFoundError, the venv may
  have been reshuffled: reinstall from `requirements.txt`
  (`uv pip install -r requirements.txt`) with Hermes closed (file locks). Confirm
  the venv Python is 3.11 (a 3.11/3.12 mismatch has broken the install before).
- **OCR needs `TESSDATA_PREFIX`** set to
  `C:\Users\Albert\AppData\Local\Programs\Tesseract-OCR\tessdata` (non-default
  install path). Only relevant for OCR ingest passes, not querying.
- **Never disable HyDE/rerank/metadata_boost to "fix" a weak answer** without
  Albert's say-so — they are tuned. A weak answer is usually a vault-coverage gap,
  not a pipeline fault.

---

## Query Patterns (how to get good answers)

This RAG is strongest when the question targets something Albert's coursework
actually covered, phrased so retrieval can find the right chunks.

### Three modes of use

**1. Recall / "what do I know about X"** — plain conceptual questions. Pulls from
notes + textbooks together, answers with citations.
- "Explain the bias-variance tradeoff from my notes."
- "What do I know about ARIMA and stationarity?"

**2. Personal-memory / "how did I do Y in my coursework"** — questions about
Albert's *own* work (homeworks, capstone, projects). Leans on the notebook/code
corpus and passed coursework. These are what the open internet cannot answer.
- "How did I use conjugate priors in my Statistics and Marketing Analytics homeworks?"
- "Show me how I implemented knowledge distillation in my capstone."

**3. Project-code lookup / "what technique do I have for Z"** — while building,
check what Albert already knows or has code for.
- "Do I have textbook code for Newton-Raphson root finding?"

### Phrasing tips
- Name the **course or context** when known ("in my Time Series course") — the
  retriever has a metadata boost that rewards course matches.
- For personal-work questions, use **"I / my"** — biases toward coursework/notebook
  chunks over generic textbook definitions.
- Cross-domain questions work ("how does the chain rule relate to backprop") but
  the answer may draw from the sibling domain — that is correct, not a miss.

### Breadth vs. precision (open tuning question)
Default `rerank_top_k = 5` (top 5 chunks reach the generator). If answers feel
narrow, raising it (e.g. 8) lets more sources through at slightly longer prompts.
This is a config value (`retrieval.rerank_top_k`), not a code change. Test 5 vs 8
by comparing real answers side by side, not by a metric.

---

## Maintenance (indexing, rebuilding, source-of-truth)

### Mental model
- **Source of truth:** the JSONL chunk files in `data\` (`chunks.jsonl` = markdown
  notes; `pdf_chunks.jsonl` = books; `lecture_chunks.jsonl`, `other_chunks.jsonl`,
  `ipynb_chunks.jsonl`, `ocr_*_chunks.jsonl`). Everything else is derived.
- **Derived, rebuildable:** ChromaDB (`data\chroma_db`, dense) and the bm25s pickle
  (`data\bm25_index.pkl`, sparse). If they look wrong, rebuild from the JSONL.
- **doc_id = sha256(source_file + text[:500])[:16]** — deterministic, so every
  ingest is idempotent (ChromaDB upserts on doc_id).

### After ANY append or metadata change, rebuild the sparse index
```powershell
python rebuild_bm25.py
```
It carries its own metadata copy and unions all chunk files.

### Recalibrating course tags (metadata-only)
```powershell
python recalibrate_courses.py            # add --dry-run first to preview
python rebuild_bm25.py                    # sync sparse metadata afterwards
```
Metadata-only because course is also baked into chunk TEXT, and doc_id hashes the
text — rewriting text would orphan every vector. The retriever's boost reads
metadata, so updating metadata is the correct, cheap lever.

### Sanity checks
```powershell
# dense count
python -c "import chromadb; print(chromadb.PersistentClient(path='data/chroma_db').get_collection('obsidian_vault').count())"
# sparse count
python -c "import pickle; d=pickle.load(open('data/bm25_index.pkl','rb')); print(len(d['metadatas']))"
```
Small dense > sparse gaps from re-appends are benign. A large mismatch or a whole
file missing from sparse means re-run `rebuild_bm25.py`.

### Recovering a broken venv
Close Hermes first (file locks), then inside the venv:
```powershell
uv pip install -r requirements.txt
```
Keep it current: `uv pip freeze > requirements.txt`.

---

## Roadmap (NOT yet built — do not assume these work)

- `quiz` mode: generate questions from a topic's chunks and drill Albert.
- `review` mode: summarize a course's key points before an exam.
- A persistent local query server so the skill answers in-flow with no reload.
- `rag.bat` launcher (activate venv + forward args).
- Canvas-file ingestion (Obsidian `.canvas` spatial concept maps) — still unbuilt.

Until built, stick to the Commands above.
