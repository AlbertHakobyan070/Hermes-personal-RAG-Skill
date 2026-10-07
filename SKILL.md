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
6. Call `POST /graph/expand` when the question is about how ideas CONNECT, and
   the corpus has a graph lane. See below — it is a separate mode, not a knob,
   and it is EXPERIMENTAL: check `GET /schema` first. A `503` here means the
   feature is switched off, not that the call was wrong — fall back to
   `/search` rather than retrying.
7. Switch to `rag-ops` for corpus or runtime mutation.

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

Read `state` on `/health`, not just `ready`. It separates the two not-ready
cases that are otherwise indistinguishable from outside:

| `state` | Meaning | Do |
|---|---|---|
| `ready` | Pipeline warm. | Proceed. |
| `loading` | Still building indexes and models. | Wait and poll. Report progress, do not diagnose. |
| `failed` | The build raised; the process stays up to say so. | Stop. Read `error` and `hint`, and hand to `rag-ops`. |

A `failed` service answers `/health` and returns 503 with the same text from
every retrieval endpoint. Treat that 503 as a diagnosis, not a transport
failure, and never retry it as if it were one.

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
| `rerank` | Override the method with a mode reported by `/schema`. `laya` is experimental and is listed even when it is off: see "Treat experimental features as off until shown on". |
| `rerank_instruction` | State the ranking criterion in plain language. Reorders the candidate pool; never filters it. |
| `lane_weights` | Reweight individual fusion lanes, e.g. `{"sparse": 1.5}`. Merges lane by lane over config and preset; every lane defaults to `1.0`. Read the lane names from `/schema`. |
| `lanes` | Restrict the call to a subset of the fusion lanes (names from `/schema`). It only restricts: a lane outside the list does not run, and a conditional lane (code, scope, `omnisearch`, `hype`) still needs its own trigger. Use it to isolate a lane while diagnosing, not as a default. |
| `metadata_boost` | Tri-state: force the course/domain/tag boost on or off for this call; omit it to follow config. |
| `gate` / `gate_threshold` / `gate_scorer` | The post-rerank relevance gate, off unless configured or forced. See "Skip weak evidence with the relevance gate". Never invent a `gate_threshold`: its units belong to the scorer. |
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

### Read a bad knob as a 200 with an error

A retrieval knob with a bad *value* does not return a 4xx, and it is never a
silent no-op. The call answers HTTP 200 and says what is wrong in the body:

- `/search`: a top-level `error`, with `results: []`.
- `/query`: `confidence:"ERROR"` with the message in `answer` (`Bad request: …`
  for a bad value, `Reranking failed: …` for a refused or failed reranker).
- `/compare`: that branch's `error`, with `retrieval_error:true`. The other
  branches still ran.

That covers an unknown `preset`, an empty or unknown `lanes` entry, a bad
`lane_weights` (unknown lane, negative weight), an unknown or refused `rerank`
mode, and a gate that cannot work. A wrong *type* (`top_k:"ten"`, `lanes:"sparse"`,
a non-numeric `gate_threshold`) or an out-of-range number is FastAPI's `422`,
raised before retrieval runs. (`/graph/expand` and `/answer` keep real `400`s.)

Read `error` and `confidence` before reading `results` or `sources`: an empty
list that carries an `error` is a rejected request, not a coverage gap. Fix the
knob and say what you changed. Do not resend the same body, and do not make the
error go away by silently dropping the knob.

### Follow the graph with `POST /graph/expand`

Some corpora carry a hand-drawn graph — nodes the user connected deliberately,
with labels saying how they relate. Graph mode walks those edges.

**This mode is EXPERIMENTAL and ships off.** `GET /schema` reports whether the
endpoint is available, which indexed lane holds the graph, and — under
`experimental.features` — the config key that switches it on. Check that before
reaching for it; a `503` is the feature being off, and the right response is to
use `/search`, not to retry or to tell the user the corpus has no graph.

Reach for it when the question is about STRUCTURE — what leads to what, what a
concept connects to, what a map covers — rather than about content that ordinary
retrieval already ranks well. Graph content usually competes perfectly well in
`/search`; this mode exists for DEPTH, following an edge the ranker would never
have surfaced.

Seed it one of two ways, never both in one call:

- `q` searches the graph lane for its own seeds.
- `seeds` takes chunk ids from a result you already have, so the walk starts
  from evidence the user has seen.

Supplying both, or neither, is a `400`. So is an unknown seed id.

No LLM runs. The response carries the nodes, the tree that was walked with each
edge's label and direction, the edges that loop back onto nodes already reached,
and `stats.truncated_at` naming the cap that stopped the walk. Report a
truncation rather than presenting a partial traversal as the whole graph.

Then decide what the nodes are for, and say which you chose:

- Take them as they are. Nothing more is needed for a structural answer.
- Send them to `POST /answer` for a grounded, cited answer over those documents
  alone.
- Send them to `POST /answer` together with the ids from a previous `/search` or
  `/query`, to answer over both stacks at once.

`POST /answer` runs generation over a document set YOU choose; no retrieval
happens. An unknown chunk id is a `400`, never a quietly smaller context.
Evidence ids that are not indexed records — live-vault excerpts and
parent-expanded sections — cannot ground an answer; drop them and say so.

Prefer offering the user the choice when a traversal is ambiguous. The mode is
built around a human decision, and silently answering from one set is the one
behaviour it exists to avoid.

### State the ranking criterion with `rerank_instruction`

Send `rerank_instruction` when the question alone does not say what makes one
chunk *better* than another — "prefer worked procedures over definitions",
"prefer primary sources over summaries". It joins the text the reranker scores
against, so it changes the order of an already-built candidate pool. It can
never add or remove a candidate. Use it instead of narrowing `q` when the topic
is right but the *kind* of material coming back is wrong.

It is honoured only by the modes that score semantically:

| `rerank` mode | Behaviour |
|---|---|
| `cross_encoder` | Applied. The model reads criterion and question together. |
| `http` | Applied. Sent as one query to the external service. |
| `lexical` | **Ignored by design.** Lexical scoring is query-term coverage; a sentence of instruction would dilute every real query term. |
| `none` | Nothing is scored at all. |
| `laya` | **Ignored by design.** The query goes into the model's own question, the wording its checkpoint was trained on. |

Never infer from the request alone that it took effect. Read the retrieval echo,
which reports both the criterion and whether it was actually applied:

```cmd
curl.exe -X POST http://127.0.0.1:8051/search -H "Content-Type: application/json" -d "{\"q\":\"confidence interval for a regression coefficient\",\"rerank_instruction\":\"prefer worked step-by-step procedures over definitions\"}"
```

- `retrieval.rerank_instruction` — the criterion that was applied, or `null`.
- `retrieval.rerank_instruction_applied` — `false` means it changed nothing.

A `false` echo under `lexical`, `none` or `laya` is correct behaviour, not a fault.
Send `""` to switch a configured criterion off for one call; omit the field
entirely to inherit the configured one.

### Skip weak evidence with the relevance gate

The gate is an optional cutoff after the rerank. Chunks scoring below
`gate_threshold` are dropped before generation (and before parent/neighbor
expansion), and when none survive `/query` **abstains**: the fixed "nothing
relevant" answer, `confidence:"LOW"`, no citations or sources, and **no LLM
call**. The echo's `retrieval.gate` says what it did:
`{enabled, scorer, threshold, kept, dropped, abstained}`, or `{"enabled":false}`
when it did not run.

- It is off unless the operator configured it. `gate:true` / `false` forces it
  for one call, and a `gate_threshold` sent alone turns it on for that call.
- **Never invent a threshold.** It is in the scorer's own units. The default
  `rerank_score` scorer reads the active reranker's score (cross-encoder logits,
  lexical coverage scores and Laya probabilities are different scales), so a
  number from one reranker means nothing for another. Use a value the operator
  gave you or the config holds; the operator picks it on the eval's dev split.
- `gate_scorer` is `rerank_score` (the default) or `laya` (experimental). A scorer
  other than the configured one needs its own `gate_threshold`.
- A gate over `rerank:"none"` is an error under the default scorer: `none` scores
  nothing, and the gate never passes unscored chunks through.
- An abstain (`retrieval.gate.abstained:true`) is the gate's verdict, not proof
  that the vault is silent: a threshold set too high produces one too. Re-run
  `/search` with `gate:false` and look at what was dropped before reporting a
  coverage gap.

### Treat experimental features as off until shown on

Some features are built and tested but unmeasured, and ship **off**: graph mode
(`graph.enabled`; `POST /graph/expand` and `POST /answer`) and the Laya reranker
(`retrieval.laya.enabled`; `rerank:"laya"` and `gate_scorer:"laya"`). Never assume
one is on, and never describe its output as better than the default: nobody has
measured that.

- `GET /schema` reports graph mode under `experimental.features`. It does not
  report Laya: `rerank_modes` lists `laya` whether or not it is enabled, and so
  does `GET /compare/options`, which marks no rerank branch unavailable. The state
  of every experimental flag is `GET :8052/api/settings`, `experimental[].on` (the
  config on disk; a `:8051` restart applies it), which `rag-ops` reads.
- A refused feature says so. `rerank:"laya"` or `gate_scorer:"laya"` with the flag
  off, the package missing or no checkpoint comes back as the error described
  above, naming the flag. It is never answered by another mode. Do not retry with
  a different one on your own: tell the user what was refused and offer the
  switch (`rag-ops`, a settings write and a `:8051` restart).
- Leave `rerank_laya` out of a `/compare` unless the flag is on: its branch fails
  alone, and the others still run.
- `rerank_instruction` is ignored by `laya`, as by `lexical` and `none`.

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
- `retrieval` for the effective preset, lanes, reranker, gate, timings, and context behavior.
- `generation` for the resolved backend, protocol, model, and usage metadata.

Present the answer, confidence, and cited labels. Mention important uncited
evidence separately. Preserve evidence IDs when the result will be compared,
audited, or revisited.

Treat `confidence:"ERROR"` as an operational result, not an answer. Preserve the
provider-aware or retrieval-aware error text. A rejected knob looks the same: see
"Read a bad knob as a 200 with an error".

### Read the retrieval echo

`retrieval` reports what actually ran. Besides the effective settings, these keys
matter when you are diagnosing or timing a call:

| Key | Read it as |
|---|---|
| `timings` | Milliseconds per stage (`hyde`, `embed`, one `lane.<name>` per lane that ran, `fuse`, `rerank`, `gate` when it ran, `total`, …). A missing stage did not run. Generation's own stages are in `generation.timings`. |
| `lanes_run` | `{lane: candidates returned}` for each lane that ran. Check it before concluding that a lane "found nothing": a lane absent here never ran. |
| `cold` | `true` means this call loaded the index or a model. Do not quote its `timings` as steady state; repeat the call. |
| `hyde_cache` | `off`, `bypass` (code intent), `hit` (draft replayed, no LLM call), `miss` (fresh draft, stored), `nocache`, or `error` (the raw query was used). |
| `gate` | The relevance gate's verdict, above. |

## Compare a tree of query branches

Use `/compare` to run one question under bounded branch overrides.

Call `GET /compare/options` first. It returns ready-to-post branch objects, so
stop hand-assembling them from `/schema` presets, `/schema` rerank modes and
`/providers`, and stop re-deriving the branch caps from prose:

```cmd
curl.exe http://127.0.0.1:8051/compare/options
```

- `branch_limits` — the real `min`, `search` and `query` caps, as enforced.
- `dimensions` — one comparable axis each (`preset`, `reranker`, `provider`),
  carrying the `mode` that axis requires and its list of branch objects.
- Post 2 or more branches from **one** dimension, with that dimension's `mode`.

Unavailable backends are **listed and marked**, not hidden. A provider branch
carries `available` and, when false, `unavailable_reason` naming the missing
credential. Never post an unavailable branch and report its failure as a
finding about the model; drop it, or say the comparison could not include it.

Read branch limits and allowed fields from `/schema` only when you need a field
`/compare/options` does not cover.

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
9. Send `rerank_instruction` when the topic is right but the kind of material is
   wrong — definitions where procedures were wanted, summaries where primary
   sources were. Confirm `rerank_instruction_applied` before judging the result.
10. Try query variants and merge distinct evidence for synonym-heavy topics.
11. Decompose into the iterative self-RAG loop when one retrieval cannot cover
    the question.
12. Stop honestly when the corpus does not contain the needed material.

Do not persistently disable tuned defaults to rescue one weak query. Use
per-call overrides, evaluate the result, and leave configuration changes to
`rag-ops`.

Do not generalise from a handful of queries. Whether a change helps across many
questions is a measurement for the eval bench (`python main.py bench run
--configs …`), which `rag-ops` owns: it runs on the dev split by default, and
only the operator opens the sealed test split.

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
`cross_encoder`, `http`, `lexical`, and `none`, plus the experimental `laya`.

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
- `laya` only when the experimental flag is on. It is refused otherwise, never
  replaced by another mode.

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
| `/health` reports `state:"failed"` | Read `error` and `hint`; hand to `rag-ops` with `GET /api/service/log`. Do not retry the 503. |
| `Reranking failed:` | Inspect `/config`; fix the reranker model, limit, device, or HTTP service through `rag-ops`. |
| `Reranking failed:` naming `retrieval.laya.enabled` | The experimental Laya reranker is off. Choose another mode explicitly, or offer the switch through `rag-ops`. Do not retry. |
| `confidence:"ERROR"` starting `Bad request:`, or an `error` on `/search` | A rejected knob, not an outage. Read the message, fix the knob, and say so. |
| `LOW` with `retrieval.gate.abstained:true` | The relevance gate dropped everything. Re-run `/search` with `gate:false` to see what it removed before reporting a coverage gap. |
| `retrieval.cold:true` | The call loaded the index or a model. Its `timings` are not steady state; repeat it. |
| Right topic, wrong kind of material | Send `rerank_instruction`; verify `rerank_instruction_applied:true`. |
| `rerank_instruction_applied:false` | The mode is `lexical`, `none` or `laya`, which ignore it. Switch to `cross_encoder` or `http`. |
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
- Running the eval bench, reviewing its question sets, and switching experimental
  features on or off.

Never mutate corpus or runtime state merely because a query is weak. Diagnose
retrieval first, then propose the smallest appropriate operation.
