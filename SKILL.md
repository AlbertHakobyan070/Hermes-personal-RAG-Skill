---
name: personal-rag
title: Albert's Personal RAG (Vault Knowledge Retrieval)
description: "Retrieve grounded, cited answers from Albert's OWN AUA coursework, lecture notes, textbooks, and notebooks. Use when he asks what he knows about a topic, how he did something in his own homework/capstone/coursework, or wants to drill his own materials for interview/exam prep. NOT for general-knowledge questions unrelated to his vault."
version: 0.2.0
author: Albert Hakobyan
license: MIT
dependencies: [chromadb, sentence-transformers, bm25s, openai, pyyaml, python-dotenv, fastapi, uvicorn, streamlit, pymupdf, pymupdf4llm]
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
Programming) plus a `swe` domain, 281 textbooks, 24 self-study tech books, lecture
PDFs, passed coursework, and his own notebooks/code. **~168,880 chunks.** Answers
are **grounded in Albert's own materials** and carry inline `[n]` citations back to
the source note/book/page.

**The core value:** when Albert asks "what do I know about X" or "how did I do Y
in my coursework," this retrieves from *his* notes — not the open internet. It is
a recall engine for his own knowledge, built to fight interview/exam imposter
syndrome and to surface his own prior code/technique during project work.

> This skill is a **playbook for invoking the RAG**, which lives as a Python
> project at `A:\DS_Vault\DS Main Vault\rag_project`. The agent drives it through
> the `terminal` toolset. It does not reimplement retrieval.

---

## ⚡ The One Rule That Matters Most

**Query through the HTTP endpoint with a single `curl`. Never drive the Streamlit
web page, and never use one-shot `python main.py query` for agent work.**

Why: the Streamlit UI (`localhost:8501`) is built for human eyeballs — an agent
"using" it has to open/snapshot/click/type the page, each a vision round-trip,
which burns a free-tier model's token budget before it ever answers. And
`python main.py query` reloads the ~350MB BM25 index on **every** call (10–30s off
the HDD). The endpoint (`serve_api.py`) loads the pipeline **once**, stays warm,
and returns compact JSON. One curl = one grounded answer. This is the whole point.

Everything below assumes the endpoint. The `main.py chat`/`query`/Streamlit paths
are documented at the end only as human fallbacks.

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
3. **Query the warm endpoint.** One curl to `POST /query`. Don't reload, don't
   screen-scrape. See the One Rule above.
4. **Don't fabricate retrieval.** Never invent citations or claim the vault
   contains something it returned no chunks for. The whole skill is trust.

---

## Environment (Windows — REQUIRED, read before any command)

This project runs **only inside the Hermes venv**. A bare PowerShell/cmd lacks the
packages and every command will fail with ModuleNotFoundError.

- **Project root:** `A:\DS_Vault\DS Main Vault\rag_project`
- **Venv:** `C:\Users\Albert\AppData\Local\hermes\hermes-agent\venv` (Python 3.11)
- **Endpoint:** `http://127.0.0.1:8051` (port 8051 — **NOT** 8000, which is Albert's Jupyter)
- **Generation proxy:** FreeLLMAPI at `http://localhost:3001` (must be running for
  answers to generate; embeddings/retrieval are local and don't need it)

Two services must be up for a query to fully succeed:
1. **FreeLLMAPI** on `:3001` (the generation backend)
2. **The RAG endpoint** on `:8051` (`serve_api.py`)

If either is down, see **Starting / checking the services** below.

---

## Querying — the endpoint (PRIMARY path for both Hermes and Claude Code)

### Health check first (always cheap, always safe)

```cmd
curl.exe http://127.0.0.1:8051/health
```

Expect `{"ready":true}`. If it doesn't respond, the endpoint isn't running — start
it (see below). If `ready` is true but queries return a generation ERROR, it's the
FreeLLMAPI proxy on `:3001` that's down.

### Ask a question (cmd.exe — Hermes' shell, and the safe default)

```cmd
curl.exe -X POST http://127.0.0.1:8051/query -H "Content-Type: application/json" -d "{\"q\":\"How did I use knowledge distillation in my capstone?\"}"
```

> **Shell escaping matters.** The Hermes agent runs **cmd.exe**, where the
> `\"`-escaped form above works. **PowerShell mangles inline JSON** (it collapses
> the quotes and you get a 422). In PowerShell use `Invoke-RestMethod` with a
> single-quoted body instead:
>
> ```powershell
> Invoke-RestMethod -Uri http://127.0.0.1:8051/query -Method Post -ContentType "application/json" -Body '{"q":"How did I use knowledge distillation in my capstone?"}'
> ```

### Response shape

```json
{
  "answer": "... text with inline [1] [2] citation markers ...",
  "confidence": "HIGH",
  "citations": [
    {"n": 1, "label": "Statistics & Inference › Bayesian inference"},
    {"n": 2, "label": "AIMA Russell & Norvig › Ch.20, p.812"}
  ],
  "sources": [
    {"n": 1, "label": "Statistics & Inference › Bayesian inference", "cited": true},
    {"n": 2, "label": "AIMA Russell & Norvig › Ch.20, p.812", "cited": true},
    {"n": 3, "label": "Wackerly Math Stats › Ch.8, p.402", "cited": false}
  ]
}
```

- `citations` = the sources the answer actually referenced (`[n]` markers), each
  with its label.
- `sources` = all retrieved sources (up to `max_sources`, default 7), each with a
  `cited` flag showing whether it made it into the answer. Surface the `cited`
  ones first; the rest are "also retrieved but not used."

If generation fails (proxy down), `/query` returns a payload with
`confidence: "ERROR"` and an `answer` explaining the backend is unreachable (the
hardened `serve_api.py` catches the failure). Relay it as "backend down," tell
Albert to start FreeLLMAPI, and don't treat it as a real answer.

---

## Starting / checking the services

### The RAG endpoint (`serve_api.py`)

The endpoint must be started once per machine session; after that it stays warm.

**Option A — `rag.bat` launcher (preferred).** A launcher in the project root that
activates the venv, cd's to the project, sets the OCR/uv env vars, and starts the
endpoint. Run:

```cmd
"A:\DS_Vault\DS Main Vault\rag_project\rag.bat" serve
```

`rag.bat` also forwards any other command to `main.py` — e.g.
`rag.bat query "..."`, `rag.bat chat`, `rag.bat eval`.

**Option B — explicit start (always works).** If no launcher exists yet, start it
by hand from cmd.exe:

```cmd
C:\Users\Albert\AppData\Local\hermes\hermes-agent\venv\Scripts\activate.bat
cd /d "A:\DS_Vault\DS Main Vault\rag_project"
python -m uvicorn serve_api:app --host 127.0.0.1 --port 8051
```

Notes that bite if ignored:
- You **must** `cd` to the project root first — uvicorn imports `serve_api` from
  the current working directory.
- The venv **must** be active (bare shell = ModuleNotFoundError).
- It loads the pipeline at startup (a few seconds), then `/health` returns
  `{"ready":true}`. Leave the window open; it's the warm server.

**Agent behaviour when the endpoint is down:** if `/health` doesn't answer, start
it with Option A or B in a background/separate terminal, wait for `/health` to go
`ready:true`, then send the query. If startup fails with ModuleNotFoundError, the
venv was reshuffled — see **Recovering a broken venv** in Maintenance. Do not fall
back to screen-scraping Streamlit.

### The FreeLLMAPI proxy (`:3001`)

```cmd
curl.exe http://localhost:3001/v1/models
```

If this fails, the proxy is down. **Tell Albert to start FreeLLMAPI** — do not try
to fix or launch it from the terminal (it's a separate app Albert manages).

---

## Host-specific notes

### Hermes
- Install by RAW URL to this single file:
  `hermes -p mer-lezun skills install <raw-github-url-to-SKILL.md>`
  (`references/` are NOT auto-fetched, which is why everything is folded into this
  one file). Per-profile; active profile is `mer-lezun`. Local-path install does
  not work.
- Hermes runs **cmd.exe** → use the `\"`-escaped `curl.exe` form above.
- Trigger: description auto-match and/or `/skill personal-rag`.

### Claude Code
- Lives as a project/personal skill (drop this `SKILL.md` into the skills
  directory Claude Code reads). It drives the same endpoint via the same `curl`.
- Claude Code typically runs in the shell you launched it from. If that's
  PowerShell, use the `Invoke-RestMethod` form; if cmd.exe, use the `curl.exe`
  form. When unsure, run the `/health` check in both styles once and use whichever
  returns clean JSON.
- Same hard rule: one `curl` to `:8051`, never the Streamlit page.

---

## How To Present Results To Albert

When you get a response, report:
1. **The answer text** (it already contains inline `[n]` markers).
2. **The confidence** the model reported (HIGH/MEDIUM/LOW/UNKNOWN).
3. **The cited sources** — each `[n]` with its `label` (course › heading, or
   book › chapter, p.N). This is what lets Albert verify against the original.

If the RAG returns nothing relevant, report that plainly. Optionally offer to
answer from general knowledge *as a clearly separate, non-grounded* answer — but
never blend the two.

If confidence is LOW/UNKNOWN or citations look thin, say so. Do not oversell a
weak grounded answer. If `confidence` is `ERROR`, the generation backend is down —
point Albert at FreeLLMAPI rather than relaying it as an answer.

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
- "How do I structure a microservices deployment with Kubernetes?" (hits the tech books)

### Phrasing tips
- Name the **course or context** when known ("in my Time Series course") — the
  retriever has a metadata boost that rewards course matches.
- For personal-work questions, use **"I / my"** — biases toward coursework/notebook
  chunks over generic textbook definitions.
- Cross-domain questions work ("how does the chain rule relate to backprop") but
  the answer may draw from the sibling domain — that is correct, not a miss.

---

## Latency & Gotchas (read before debugging)

- **Always query the endpoint, never the Streamlit page or `main.py query`.** See
  the One Rule. Screen-scraping burns tokens; one-shot query reloads 350MB.
- **`/health` not responding** = endpoint down → start `serve_api.py` (Option A/B
  above). **`confidence: ERROR`** = FreeLLMAPI proxy down → tell Albert to start it.
- **Wrong port.** The endpoint is **8051**, not 8000 (Jupyter) and not 8501
  (Streamlit). Hitting the wrong port is a common self-inflicted "it's down."
- **PowerShell mangles inline JSON.** Use `Invoke-RestMethod -Body '...'`
  (single-quoted) in PowerShell; the `\"`-escaped `curl.exe` form is for cmd.exe.
- **A Hermes app reinstall can wipe venv packages** (it has before — chromadb,
  sentence_transformers, and an opentelemetry/tokenizers version clash). If a
  command dies with ModuleNotFoundError, the venv was reshuffled: see Maintenance.
- **Never disable HyDE/rerank/metadata_boost to "fix" a weak answer** without
  Albert's say-so — they are tuned. A weak answer is usually a vault-coverage gap,
  not a pipeline fault.

---

## Maintenance (indexing, rebuilding, source-of-truth)

These are admin/ingest tasks — run them in a normal venv shell, not via the
endpoint. They are not part of the query path.

### Mental model
- **Source of truth:** the JSONL chunk files in `data\` (`chunks.jsonl` = markdown
  notes; `pdf_chunks.jsonl` = books; `lecture_chunks.jsonl`, `other_chunks.jsonl`,
  `ipynb_chunks.jsonl`, `ocr_*_chunks.jsonl`, `tech_chunks.jsonl`). Everything
  else is derived.
- **Derived, rebuildable:** ChromaDB (`data\chroma_db`, dense) and the bm25s pickle
  (`data\bm25_index.pkl`, sparse). If they look wrong, rebuild from the JSONL.
- **doc_id = sha256(source_file + text[:500])[:16]** — deterministic, so every
  ingest is idempotent (ChromaDB upserts on doc_id).
- **Restart the endpoint after re-indexing** — `serve_api.py` loads the index at
  startup and stays warm, so it won't see new chunks until restarted.

### Add new content (incremental)
```cmd
:: notebooks / code (.ipynb/.py/.R/.Rmd)
python main.py ingest-notebooks
python main.py index --append data\ipynb_chunks.jsonl

:: PDFs (books / lectures / coursework — scoped passes)
:: NOTE: --skip-books is TOKEN-based — a folder named "...Books" counts as a book
::       folder. For e.g. "Tech Books" use --only-books, not --skip-books.
python main.py ingest-pdfs --skip-books --include-path "Current Courses"
python main.py index --append data\lecture_chunks.jsonl

:: after ANY ingest, rebuild the sparse index, then restart the endpoint:
python rebuild_bm25.py
```
> Don't trust the "Next:" hint printed after an ingest — it's hardcoded to
> `pdf_chunks.jsonl` regardless of `--output`. Append the file you actually produced.

### Recalibrating course tags (metadata-only)
```cmd
python recalibrate_courses.py            :: add --dry-run first to preview
python rebuild_bm25.py                   :: sync sparse metadata afterwards
```
Metadata-only because course is also baked into chunk TEXT, and doc_id hashes the
text — rewriting text would orphan every vector. The retriever's boost reads
metadata, so updating metadata is the correct, cheap lever.

### Sanity checks
At ~168K chunks, any full-collection ChromaDB `get`/`delete` hits SQLite's "too
many SQL variables" cap — **paged** helper scripts exist (`check_meta.py`,
`check_techbooks.py`, `delete_doc.py`); any new ChromaDB script must page in
~5000-row batches too.
```cmd
:: sparse count
python -c "import pickle; d=pickle.load(open('data/bm25_index.pkl','rb')); print(len(d['metadatas']))"
:: tech-book / domain breakdown (paged)
python check_techbooks.py
```
A small dense > sparse gap (dense reads ~168,880, sparse ~167,727) is benign —
BM25 dedupes by doc_id across the union. A whole file missing from sparse means
re-run `rebuild_bm25.py`.

### Recovering a broken venv
Close Hermes first (file locks from ~8 running venv python processes — kill them:
`Get-Process python | ? {$_.Path -like "*hermes*"} | Stop-Process -Force`), then
inside the venv:
```cmd
uv pip install -r requirements.txt
```
Keep it current: `uv pip freeze > requirements.txt`. Confirm the venv Python is
3.11 (a 3.11/3.12 mismatch has broken the install before).

---

## Human fallback paths (NOT for agents)

These exist for Albert at a keyboard. An agent should use the endpoint instead.

- **Interactive terminal session (warm):** `python main.py chat` — loads once,
  answers in a loop. Good for Albert drilling himself directly.
- **Web app (warm, richest view):** `python -m streamlit run app.py` →
  `localhost:8501`, answer + confidence badge + expandable cited sources. Use
  `python -m streamlit`, **NOT** bare `streamlit` and **NOT** `python main.py serve`
  (both fail with WinError 2 — streamlit isn't on PATH).
- **One-shot (cold, slow):** `python main.py query "..."` — reloads the index each
  call. Only for a single isolated human question.
- **Eval:** `python main.py eval` — 30 golden questions, prints recall / retrieval
  hit / confidence / speed. For a reproducible number, pin
  `generation.model: "llama-3.3-70b-versatile"` in config.yaml, run, then revert
  to `"auto"` (on auto the number wobbles because the router serves different
  models per run). Current baseline: ~80.8% keyword recall, 93.3% retrieval hit.

---

## Current config state (for reference)
- `generation.model: "auto"` (FreeLLMAPI auto-fallback chain)
- `retrieval.rerank_top_k: 7` (raised from 5 for richer citations)
- `verify_citations: false` (kept off — it narrows answers; we want breadth)
- Embeddings: local `bge-small-en-v1.5`, 384-dim, CPU
- Domains include `swe` (added with the tech-book ingest)

---

## Roadmap (NOT yet built — do not assume these work)

- `quiz` mode: generate questions from a topic's chunks and drill Albert.
- `review` mode: summarize a course's key points before an exam.
- Telegram bot hitting the same `POST /query` endpoint.
- Canvas-file ingestion (Obsidian `.canvas` spatial concept maps) — files are
  fixed and ready; loader still unbuilt.

Until built, stick to the Commands above.
