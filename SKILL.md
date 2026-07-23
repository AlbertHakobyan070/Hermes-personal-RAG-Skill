---
name: personal-rag
description: Retrieve grounded, cited answers from a configured Obsidian-vault RAG; inspect raw evidence; compare retrieval presets, rerankers, and generation providers; and diagnose weak or failed queries. Use for questions about the user's indexed notes, books, coursework, notebooks, or code, including multi-step synthesis and retrieval-quality investigation. Do not use for unrelated general knowledge or corpus mutation; use rag-ops for ingest, OCR, metadata, deletion, settings, and service operations.
---

# Personal RAG

Drive the warm query API on `http://127.0.0.1:8051` through JSON. Prefer the
API to browser automation or a cold one-shot CLI. Keep every substantive claim
grounded in returned evidence, preserve citation labels, and report coverage
gaps instead of filling them with unmarked general knowledge.

## Follow the decision ladder

Choose the cheapest adequate operation:

1. Call `GET /health` to confirm that the warm API is ready.
2. Call `GET /schema`, `GET /config`, and `GET /providers` to discover current
   capabilities instead of assuming preset, reranker, or provider names.
3. Call `POST /search` for local retrieval without final answer generation.
   Use it to inspect evidence, tune retrieval, or answer from chunks yourself.
4. Call `POST /query` for a generated, cited answer after retrieval looks sound.
5. Call `POST /compare` only when branch-level comparison will clarify a real
   retrieval or generation decision.
6. Switch to `rag-ops` for corpus or runtime mutation.

If generation fails, repeat the intent through `/search` and reason over the
returned chunks. Do not abandon usable local retrieval because a provider is
unavailable.

## Start and probe the service

Activate the project's configured environment, enter its root, and prefer its
launcher:

```cmd
rag serve
```

Use an explicit start only when the launcher is unavailable:

```cmd
python -m uvicorn serve_api:app --host 127.0.0.1 --port 8051
```

Probe readiness and discovery:

```cmd
curl.exe http://127.0.0.1:8051/health
curl.exe http://127.0.0.1:8051/schema
curl.exe http://127.0.0.1:8051/config
curl.exe http://127.0.0.1:8051/providers
curl.exe http://127.0.0.1:8051/stats
```

Read `/stats` for live corpus totals. Never quote a count copied into this
playbook.

## Send JSON safely

Use escaped JSON with `curl.exe` in `cmd.exe`:

```cmd
curl.exe -X POST http://127.0.0.1:8051/query -H "Content-Type: application/json" -d "{\"q\":\"Explain regularization from my notes\"}"
```

Use `Invoke-RestMethod` with a single-quoted body in PowerShell:

```powershell
Invoke-RestMethod -Uri http://127.0.0.1:8051/query -Method Post `
  -ContentType 'application/json' `
  -Body '{"q":"Explain regularization from my notes"}'
```

## Control retrieval deliberately

Read the live schema before setting optional fields. Use these fields as the
core control surface:

| Field | Use |
|---|---|
| `q` | Supply the required question. |
| `preset` | Select a configured retrieval bundle discovered from `/schema` or `/config`. |
| `auto_preset` | Leave `true` for intent-based preset selection; send `false` for a config-only baseline. |
| `top_k` | Control how many reranked chunks survive. |
| `dense_top_k` / `sparse_top_k` | Widen vector/BM25 candidate pools before fusion and reranking. |
| `hyde` | Force hypothetical-answer expansion on or off. |
| `hype` | Add the hypothetical-question lane when its index exists. |
| `omnisearch` | Fuse live-vault results when the configured Obsidian service is available. |
| `parent_context` | Replace a matched note chunk with its full section. |
| `neighbor_context` | Add adjacent PDF pages as supplementary context. |
| `rerank` | Override the method with a mode reported by `/schema`. |
| `include_text` | Request an evidence text excerpt. |
| `max_sources` | Cap returned sources without changing retrieval itself. |
| `retrieve_only` | Skip generation on `/query` and return evidence. |
| `provider` / `model` | Select a configured generation backend/model for `/query`; never inject an endpoint or secret. |
| `max_tokens` | Bound answer generation, not retrieval. Avoid truncating the citation/confidence footer. |

Treat optional booleans as tri-state controls: omit them to preserve
config/preset behavior, send `true` to force them on, and send `false` to force
them off. Do not serialize an unchecked UI control as `false` unless that is the
intended override.

Use `auto_preset:false` when constructing a true baseline. An omitted preset can
otherwise auto-select a code-oriented bundle when the query looks like a code
request.

Discover common preset behavior live. Typically:

- Choose a concept-oriented preset for tight definitions.
- Choose a code-oriented preset for exact implementations and scripts.
- Choose a synthesis-oriented preset for broad, parent/neighbor-expanded
  answers.
- Omit `preset` when automatic routing is desirable.

Name the domain, course, content type, library, or user tag in the query when it
matters. Let scope routing and tag boosts act on explicit wording.

## Inspect evidence before generation

Use `/search` for cheap tuning and agent-side reasoning:

```cmd
curl.exe -X POST http://127.0.0.1:8051/search -H "Content-Type: application/json" -d "{\"q\":\"conjugate priors in my homework\",\"preset\":\"concept\",\"include_text\":1200}"
```

Read each source as an evidence record:

- Preserve `id` as the effective evidence identity used for comparisons.
- Preserve `origin_id` as the original indexed or live-retrieval identity.
- Call `GET /chunks/{id}` only when `lookup_available` is `true`.
- Expect parent-expanded evidence to use an effective `parent:<id>` identity
  while retaining the matched child in `origin_id`.
- Expect live-vault evidence to use an effective `live:<hash>` identity and
  report `lookup_available:false`.
- Treat `score` as useful only within a compatible ranking method.

Fetch a lookup-capable record without searching again:

```cmd
curl.exe "http://127.0.0.1:8051/chunks/<evidence-id>?include_text=2000"
```

Do not dereference a live evidence ID. Use its label, path metadata, and excerpt
from the original response.

## Generate and present a cited answer

Call `/query` after verifying that retrieval is relevant:

```cmd
curl.exe -X POST http://127.0.0.1:8051/query -H "Content-Type: application/json" -d "{\"q\":\"How do my notes connect bias and variance?\",\"preset\":\"synthesis\"}"
```

Inspect:

- `answer` for inline citation markers.
- `confidence` for the model's groundedness assessment.
- `citations` for sources actually cited.
- `sources` for all evidence shown to generation.
- `retrieval` for the effective preset, lanes, reranker, and context behavior.
- `generation` for the resolved backend, protocol, model, and usage metadata.

Present the answer, confidence, and cited labels. Mention important uncited
evidence separately. Preserve evidence IDs when the result will be compared,
audited, or revisited.

Treat `confidence:"ERROR"` as an operational result, not an answer. Preserve the
provider-aware or retrieval-aware error text.

## Compare a tree of query branches

Use `/compare` to run one question under bounded branch overrides. Read branch
limits and allowed fields from `/schema`.

Compare retrieval without spending generation tokens:

```cmd
curl.exe -X POST http://127.0.0.1:8051/compare -H "Content-Type: application/json" -d "{\"q\":\"How did I use distillation?\",\"mode\":\"search\",\"branches\":[{\"id\":\"baseline\",\"label\":\"Config baseline\",\"auto_preset\":false},{\"id\":\"concept\",\"preset\":\"concept\"},{\"id\":\"lexical\",\"rerank\":\"lexical\"}]}"
```

Use query mode only when comparing generated answers:

```cmd
curl.exe -X POST http://127.0.0.1:8051/compare -H "Content-Type: application/json" -d "{\"q\":\"Summarize my approach\",\"mode\":\"query\",\"branches\":[{\"id\":\"default\",\"label\":\"Default provider\"},{\"id\":\"alternate\",\"label\":\"Alternate provider\",\"provider\":\"<configured-provider>\"}]}"
```

Compare:

- Effective evidence membership and ranks by `id`.
- Original indexed/live membership and ranks by `origin_id`.
- Common and branch-unique evidence.
- Pairwise overlap and rank spread.
- Answer text, confidence, citations, and generation provenance in query mode.

Never compare raw score magnitudes across cross-encoder, HTTP, lexical, or fused
order branches; their scales are incompatible. Prefer membership, rank, and
grounded answer differences.

Keep retrieval overrides identical for provider-only branches. Treat exact
evidence reuse as an API contract: branches that differ only by provider/model
must reuse one retrieval result. Verify the contract in the response before
attributing answer differences to generation:

1. Compare each branch's ordered `sources[].id` array and require exact equality.
2. Compare each branch's `retrieval` echo and require equality for every
   retrieval-affecting field.
3. Treat any mismatch as evidence drift or a contract violation; report it and
   do not claim a provider-only comparison.

## Use iterative ReAct/self-RAG for multi-hop questions

Switch from one-shot querying when the request asks to compare, connect, trace,
collect, synthesize across documents, or explain a result whose first evidence
set is visibly incomplete.

1. Decompose the request into retrieval-sized subquestions.
2. Run `/search` for each subquestion with the most suitable preset and scope
   wording.
3. Read the chunks and identify unsupported claims or missing links.
4. Turn each gap into another targeted `/search` using a new phrase, scope,
   synonym, context expansion, or HyPE setting.
5. Stop when another search adds no material evidence or the vault clearly lacks
   a required leg.
6. Check every drafted claim against a returned source.
7. Remove unsupported claims or label any necessary general knowledge as
   separate and ungrounded.

Prefer a few focused retrieval rounds over repeatedly generating whole answers.

## Tune toward relevant, well-cited evidence

Climb this ladder only while each step adds value:

1. Run a baseline `/search` or `/query`.
2. Name the intended scope or user tag explicitly in `q`.
3. Select the preset that matches concept, code, or synthesis intent.
4. Raise `dense_top_k` and/or `sparse_top_k` when the expected source never
   enters the candidate set.
5. Raise `top_k` when the expected source appears but falls below the final
   cutoff.
6. Force `hyde:true` for paraphrased conceptual questions; try `hyde:false` for
   code and exact-term hunts when expansion adds prose bias.
7. Try `hype:true` when user wording differs from document wording.
8. Add `parent_context` or `neighbor_context` when evidence is relevant but
   fragmentary.
9. Try query variants and merge distinct evidence for synonym-heavy topics.
10. Decompose into the iterative self-RAG loop when one retrieval cannot cover
    the question.
11. Stop honestly when the corpus does not contain the needed material.

Do not persistently disable tuned defaults to rescue one weak query. Use
per-call overrides, evaluate the result, and leave configuration changes to
`rag-ops`.

## Use Omnisearch, tags, and HyPE correctly

Use `omnisearch:true` to fuse currently unindexed vault notes into normal
retrieval. Use the raw passthrough only for a direct live-vault lookup:

```cmd
curl.exe "http://127.0.0.1:8051/omnisearch?q=<url-encoded-query>&k=5"
```

Report an unavailable live service explicitly. Do not imply that indexed
retrieval is also unavailable.

Include known tags naturally in `q` so metadata boosts can act. Use `hype:true`
only as an additional lane; expect partial coverage when hypothetical questions
were built for only part of the corpus.

## Diagnose reranking without hiding failures

Read the active method, model, and maximum length from `GET /config`. Discover
valid per-call modes from `/schema`; current implementations expose
`cross_encoder`, `http`, `lexical`, and `none`.

Treat messages beginning with `Reranking failed:` as retrieval failures, not
generation-provider outages. Preserve the underlying model, transport, status,
or response-schema error. Do not silently fall back.

Keep `BAAI/bge-reranker-base` at or below its 512-token input limit. Treat a
larger configured `cross_encoder_max_length` as invalid even if another model
supports more context. Expect large cross-encoders to load slowly on CPU.

Use:

- `cross_encoder` for semantic ranking when the configured model is available.
- `lexical` for fast exact-term ranking without model loading.
- `none` to inspect fused order deliberately.
- `http` only when the configured external `/v1/rerank` service is reachable.

## Diagnose generation providers

Call `GET /providers` before selecting a backend. Use only registry aliases and
models exposed by the service. Never send a base URL, environment-variable
value, or API key through `/query` or `/compare`.

Interpret provider metadata carefully:

- Treat `key_present` as presence only.
- Treat `key_compatible` as credential-type compatibility.
- Treat `available` as configured readiness, not proof of remote reachability.
- Establish real reachability with a generated query.

For the configured MiniMax M3 Token Plan backend, use the subscription/credits
credential beginning `sk-cp-`. Reject a pay-as-you-go credential beginning
`sk-api-` for Token Plan quota. Expect the registry entry to use the declared
MiniMax M3 model and protocol; discover the exact live values from `/providers`.
Store or replace the secret only through the permission-gated `rag-ops`
provider-key workflow.

## Triage common symptoms

| Symptom | Next action |
|---|---|
| Connection refused on `:8051` | Start the warm API and poll `/health`. |
| `/health` is ready but generation fails | Inspect `/providers`; retry retrieval through `/search`. |
| `Reranking failed:` | Inspect `/config`; fix the reranker model, limit, device, or HTTP service through `rag-ops`. |
| Expected source absent from `/search` | Widen candidate pools, name scope/tags, try HyPE, and rephrase. |
| Relevant source appears below cutoff | Raise `top_k`. |
| Relevant evidence is fragmentary | Enable parent/neighbor context or use a synthesis preset. |
| Code query returns lecture prose | Use a code preset, explicit code wording, or `hyde:false`. |
| Answer truncates before citations/confidence | Raise or omit `max_tokens`. |
| High confidence but irrelevant citations | Treat retrieval as wrong; retune instead of trusting the label. |
| Live evidence cannot be fetched from `/chunks` | Respect `lookup_available:false`; use the returned excerpt/path. |
| All tuned branches stay off-topic | Report a corpus-coverage gap and offer `rag-ops` ingest separately. |

## Respect the operations boundary

Keep query-time reads and bounded comparisons in this skill. Use `rag-ops` for:

- Ingest, OCR, index append/rebuild, and HyPE construction.
- Retagging or deleting indexed documents.
- Persistent settings and provider-secret changes.
- Vault switching and service restart.
- Long-running job monitoring and corpus integrity checks.

Never mutate corpus or runtime state merely because a query is weak. Diagnose
retrieval first, then propose the smallest appropriate operation.
