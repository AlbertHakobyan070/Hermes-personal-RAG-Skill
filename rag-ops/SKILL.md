---
name: rag-ops
description: "Operate and diagnose a configured personal-RAG corpus through its management API: inspect documents and settings, ingest files, run OCR and indexing jobs, manage tags, remove indexed documents, switch vaults, store declared provider credentials, restart the warm query service, run the retrieval eval bench and review its questions, and monitor failures. Use for corpus or runtime operations, not for answering a question from the vault; use personal-rag for retrieval, cited answers, and query comparisons."
---

# RAG Operations

Drive the management API on `http://127.0.0.1:8052` through JSON. Use the query
API on `http://127.0.0.1:8051` only for read-side diagnostics and verification.
Prefer API calls to browser automation.

## Enforce permission tiers

Call `GET /api/schema` before acting. Honor the permission tier attached to each
endpoint and job kind:

| Tier | Obligation |
|---|---|
| `read` | Execute when relevant without mutation approval. |
| `mutating` | State the exact intended change and obtain confirmation before executing. |
| `destructive` | Resolve exact targets and chunk counts, explain that deletion affects the index, and obtain explicit confirmation. |

Treat provider-secret writes, settings changes, uploads, ingest, OCR, retagging,
vault switching, restarts, and index jobs as mutations even when reversible.
Never broaden a confirmation from one target or operation to another.

Never delete source notes, notebooks, or PDFs. Use document deletion only to
remove indexed records and derived data.

## Start and discover the console

Activate the project's configured environment, enter its root, and start:

```cmd
rag console
```

Use an explicit start only when the launcher is unavailable:

```cmd
python -m uvicorn manage_api:app --host 127.0.0.1 --port 8052
```

Probe the management and query surfaces:

```cmd
curl.exe http://127.0.0.1:8052/api/schema
curl.exe http://127.0.0.1:8052/api/overview
curl.exe http://127.0.0.1:8051/health
curl.exe http://127.0.0.1:8051/config
curl.exe http://127.0.0.1:8051/providers
curl.exe http://127.0.0.1:8051/schema
```

Read live schemas instead of copying endpoint limits, job parameters, provider
names, counts, or preset names into this playbook.

## Inspect before changing

Use the read layer to resolve exact state:

```cmd
curl.exe http://127.0.0.1:8052/api/overview
curl.exe "http://127.0.0.1:8052/api/documents?q=<term>&limit=20"
curl.exe http://127.0.0.1:8052/api/facets
curl.exe "http://127.0.0.1:8052/api/vault/search?q=<url-encoded-query>"
curl.exe "http://127.0.0.1:8052/api/documents/preview?source_file=<source-key>&n=3"
curl.exe http://127.0.0.1:8052/api/jobs
```

Use exact `source_file` keys returned by `/api/documents` for preview, retag, and
delete operations. Never reconstruct them from display labels.

Use `GET /api/vault/tree` and `/api/vault/search` to inspect vault-side files
without assuming a filesystem root. Use `/api/facets` to discover live domains,
courses, tags, and content types.

## Diagnose query behavior from the operations side

Read:

- `GET :8051/config` for effective retrieval defaults, active reranker model,
  device, mode, maximum length, and generation provenance.
- `GET :8051/providers` for configured generation aliases and key readiness
  without secret values.
- `GET :8051/schema` for current presets, rerank modes, comparison limits, and
  request fields.
- `GET :8051/chunks/{id}` for a stable evidence record when
  `lookup_available:true`.
- The `retrieval` echo of a targeted `/search` for what actually ran: `timings`
  and `cold` (a cold call's timings are a one-off load, not steady state),
  `lanes_run`, `hyde_cache`, and `gate` (what the relevance gate dropped).
- `POST :8051/compare` through `personal-rag` for evidence membership/rank and
  provider comparisons.

Distinguish `sources[].id` from `origin_id`. Treat `id` as effective comparison
identity and `origin_id` as indexed/live origin. Expect parent evidence to use
`parent:<id>` and live evidence to be non-dereferenceable.

Never compare raw reranker score magnitudes across methods.

## Manage provider credentials safely

Prefer the `:8052` console's Settings UI for a person entering or replacing a
provider credential. Keep the value out of shell history, process arguments,
terminal transcripts, and agent-visible logs.

Use `POST /api/providers/key` only from a trusted API client. Take the allowed
environment-variable name from `GET :8051/providers`, confirm the mutation, and
treat this as a placeholder request shape rather than a command containing a
real value:

```json
{
  "env": "<declared-api-key-env>",
  "value": "<secret-from-protected-input>"
}
```

Require automation to obtain the value from protected standard input or an
approved secret store, construct the JSON body in memory, submit it directly,
and avoid echoing or persisting the body. Never interpolate a real credential
into an inline curl command.

Use the endpoint's documented clearing form when removing a credential. Never
write a secret into YAML, source control, command logs, skill text, or an
undeclared environment-variable name. Never attempt to read a stored value back;
inspect only presence and compatibility.

For a configured MiniMax M3 Token Plan backend, require the subscription/credits
credential beginning `sk-cp-`. Reject a pay-as-you-go credential beginning
`sk-api-` for Token Plan quota. Preserve prefix-validation errors instead of
bypassing them.

Restart the warm query API after a credential change so it reloads environment
state.

## Change settings deliberately

Read editable fields, current values, allowed choices, and restart requirements:

```cmd
curl.exe http://127.0.0.1:8052/api/settings
```

Confirm the mutation, then send only intended dotted keys to
`POST /api/settings`. Do not copy a stale whole-settings payload over concurrent
changes. Respect each field's reported restart policy.

Preflight a cross-encoder before writing `retrieval.cross_encoder_model`.
Switching reranker is a config write plus a `:8051` restart, and every way it
can go wrong used to look identical from the console — the restart simply never
reported ready:

```cmd
curl.exe -X POST http://127.0.0.1:8052/api/rerank/check -H "Content-Type: application/json" -d "{\"model\":\"BAAI/bge-reranker-v2-m3\"}"
```

It downloads nothing. Read:

- `usable` — the verdict. Do not persist a model that reports `false`.
- `reachable` / `cached` — whether the hub answers, and whether the weights are
  already local. An uncached model means the next restart pays a download.
- `context_length` vs `max_length` — a configured maximum above the model's own
  limit is a guaranteed failure at query time, not a slow path.
- `warnings` — act on these before persisting.

A model reported unreachable is not automatically missing. A stale or revoked
Hugging Face token makes the hub answer **401**, which surfaces as
"repository not found" for *every* public model; this endpoint distinguishes
that case, so read its explanation instead of concluding the model was renamed.

Validate reranker combinations before persisting:

- Keep `BAAI/bge-reranker-base` at or below its 512-token context limit.
- Use a model's declared hard context limit, not a limit copied from another
  model.
- Configure an HTTP URL/model before choosing `http` reranking.
- Treat invalid model/limit/device combinations as explicit failures.

Set `retrieval.rerank_instruction` only as a durable, corpus-wide ranking
criterion. A criterion for one question belongs in the per-call field that
`personal-rag` owns, not in persisted configuration. It is ignored by
`rerank_mode: lexical`, `none` and `laya`.

Use `lexical` for a model-free low-power profile and `none` only for deliberate
fused-order diagnostics. Do not silently downgrade a failed configured
reranker.

## Switch experimental features on deliberately

Some features are built and tested but unmeasured, so they ship **off**.
`GET /api/settings` lists them under `experimental`: each row carries an `id`, the
config `key` that switches it, a `label`, `what` it does, `why_off`, what stays
`unaffected`, its `surfaces`, and whether it is `on` in the config on disk. Read
`what` and `why_off` to the operator before proposing a switch.

- The flags (`graph.enabled`, `retrieval.laya.enabled`) are ordinary dotted keys:
  confirm, then send them through `POST /api/settings` in the form
  `GET /api/settings` lists. Nothing hot-applies; the response names what to
  restart (`:8051`, and graph mode also reloads the console page). Verify
  afterwards instead of assuming.
- **Laya also needs** the optional `laya` package (deliberately not in
  `requirements.txt`; the refusal message carries the exact pinned install command)
  and a checkpoint under `retrieval.laya.model_dir`;
  `docs/laya-finetune.md` is the manual. Without them the mode is refused with an
  error naming what is missing, never answered by another reranker. The flag alone
  does not prove it works: finish with a targeted `/search` using `rerank:"laya"`
  and read the result.
- **The relevance gate has no console switch.** Its keys are
  `retrieval.relevance_gate.{enabled, scorer, threshold}` in `config.yaml`. An
  enabled gate with no threshold, or an unknown scorer, is a startup error by
  design (`/health` reports `failed`, with the reason). Do not enable it until the
  operator has chosen a threshold on the eval's dev split; the per-call `gate` and
  `gate_threshold` fields on the query API let it be exercised without persisting
  anything.
- Experimental means unmeasured. Do not present either feature as an improvement.

## Queue and monitor jobs

Read `job_kinds` from `/api/schema`. Confirm mutating work, then enqueue one
specific job:

```cmd
curl.exe -X POST http://127.0.0.1:8052/api/jobs -H "Content-Type: application/json" -d "{\"kind\":\"<job-kind>\",\"params\":{}}"
```

Use the schema-defined parameters for common operations such as:

- PDF, notebook, code, Markdown, canvas/graph, or URL ingest.
- Index append or full rebuild.
- BM25 rebuild.
- Scoped HyPE construction.
- Metadata recalibration.
- Retrieval evaluation (the golden-suite `eval` job kind; the bench is a command
  line tool, not a job kind, see below).

Avoid a full-corpus re-embed or unscoped HyPE build without explicit approval
and a reason supported by current corpus state. Prefer scoped append and
metadata operations.

Poll and tail the returned job ID:

```cmd
curl.exe http://127.0.0.1:8052/api/jobs/<job-id>
curl.exe "http://127.0.0.1:8052/api/jobs/<job-id>/log?offset=0"
```

Continue until the job reaches a terminal state. Read the log before retrying.
Use retry or cancel only through the corresponding schema-documented endpoint
and permission tier.

Report the job kind, target, terminal state, relevant counts, and any unresolved
error. Never describe queued work as complete.

## Ingest a graph lane deliberately

Canvas ingest is not another file type, and the console does not file it as
one. Two read-only endpoints exist because a graph needs answering questions no
other lane does:

```cmd
curl.exe http://127.0.0.1:8052/api/canvas/folders
curl.exe -X POST http://127.0.0.1:8052/api/canvas/preview -H "Content-Type: application/json" -d "{\"include_path\":\"<folder>\"}"
```

`folders` reports which vault trees hold canvases, with counts — scope from that
rather than guessing a path substring. `preview` runs the real loader into a
scratch file, indexes nothing, and reports the chunk and edge counts, the share
of chunks that come out carrying edges, the context-inflation cost, and one
composed sample. Read a preview before queueing an ingest, and report its
numbers rather than predicting them.

Two things to get right:

- **Stamp the metadata.** Canvas filenames rarely carry a course keyword, so an
  unscoped run lands chunks as the generic domain. The preview says so
  explicitly when it would happen. Set `force_domain` / `force_tags`.
- **A scoped run needs its own output file.** The loader truncates whatever it
  writes, so a folder-scoped run pointed at the canonical canvas chunk file
  would shrink it to that folder: the dense index keeps every chunk because
  append upserts, while the sparse index is re-derived from the chunk files and
  loses the rest. The job builder refuses that combination; give a scoped run
  its own file.

`context_depth` above `0` inlines a neighbour's text into each chunk so the
chunk is self-contained. It duplicates text across chunks and inflates the
index. Treat it as a re-ingest decision with a measured cost — the preview and
the ingest log both report the share — not a default worth flipping on.

## Measure retrieval with the eval bench

The bench scores labelled questions per named pipeline configuration. It is a
command-line tool, not a console job: run `rag bench …` (the launcher forwards
what it does not know to `python main.py bench …`) in the project environment. It
builds its own pipeline in-process, so on a small machine do not run it beside the
warm query API, OCR, or another heavy job.

| Subcommand | Does | State it touches |
|---|---|---|
| `bench sample` | Draws seed packs for drafting questions. | Writes `eval/seeds/`. |
| `bench draft --suite S [--n N]` | Drafts questions from that suite's seed pack with an LLM (provider `freellmapi` unless `--provider`). The model writes the question and its nuggets; the gold locator is read off the seed chunk, never written by the model. Resumable. | Appends to `eval/sets/<suite>.yaml`. Calls the provider, and the warm query API (`/search`, `/chunks/{id}`) for twins and for the unanswerable check. |
| `bench validate [--write-cache]` | Checks the question sets against the corpus. | Reads only; `--write-cache` writes `<sets>/.review_cache.json` (whole sets directory, so not with `--suites`). |
| `bench split` | Assigns `dev` / `test` to questions that have none, once. | Rewrites the sets files it changes. |
| `bench run --configs … [--split …] [--limit N]` | Scores configurations: `ladder`, `loo`, `factorial`, or comma-separated names from `eval/configs.yaml`. | Creates `eval/runs/<id>/`; a sealed split also appends to `eval/test_ledger.jsonl`. |
| `bench report <run_dir>` | Re-renders a finished run's `report.md`. | Rewrites that file. |

Rules that matter:

- **Confirm before anything that writes.** State what it will do, then proceed on
  approval. That includes `bench run`, which is slow and RAM-heavy;
  `--configs factorial` is the largest, so prefer a `--limit` smoke run first.
- **A draft is not a verified question.** A drafted record keeps
  `provenance.status: draft` until the operator verifies it in the console's Eval
  tab. Questions the operator writes carry `author: owner` and are reported apart
  from `author: draft`. A suite a draft run left short is topped up by running it
  again (seeds lost to an LLM error are asked again), then by `bench sample` for
  fresh seeds.
- **The test split is sealed.** Use `--split dev`, the default. Never pass
  `--unseal-test`, or `--split test` / `all`, without the operator's explicit,
  specific approval: every opening is logged to the ledger, and the test split is
  meant to be looked at once, after configurations and thresholds were chosen on
  dev. Never edit or delete the ledger or a run record, and never re-run to chase a
  better test number.
- **Exit codes.** `2` is a mistake in the command (`ERROR: …` on stderr: unknown
  suite, missing sets directory, sealed split, nothing to run). `1` means question
  rows failed (read each `error` in `per_query.jsonl`), or, for `bench validate`,
  that a gold file or locator does not resolve. Never describe a partial run as
  complete.
- **Read the report as written.** A ladder step counts only when its paired 95%
  interval excludes zero; a suite marked *diag* is a diagnostic; cold rows are left
  out of the latency percentiles. A configuration that needs an experimental
  feature (`laya-rerank`, `gate-laya`) fails loudly while the feature is off.
- **Run records and question sets quote the vault.** Summarise them for the
  operator; do not paste their text elsewhere.

The review queue is on this API, with the usual tiers. `GET /api/eval/progress`,
`GET /api/eval/questions` (filters `suite`, `status`, `split`) and
`GET /api/eval/questions/{qid}` are `read`. `POST /api/eval/questions/{qid}` with
`{"action": "verify" | "reject" | "edit"}` is `mutating`: it rewrites that suite's
sets file. Verifying is the operator's check that a drafted question is sound, so
do not verify, reject or edit on their behalf beyond the exact question and action
they named. An edit sends the whole replacement `record`, is validated by the same
schema the bench loads with, and cannot change `id`, `suite` or `split`. These
endpoints answer with real status codes (`400`, `404`), unlike the query API's
retrieval knobs.

## Handle uploads and the inbox

Inspect `/api/inbox` before ingesting. Upload only user-approved files:

```cmd
curl.exe -X POST http://127.0.0.1:8052/api/upload -F "files=@<local-file>"
```

Use `POST /api/ingest_inbox` with schema-defined metadata, chunking, OCR, and
conflict options. Treat a conflict response as a request to resolve possible
duplicates, not as permission to force ingest.

Use `/api/inbox/delete` only for confirmed inbox cleanup. Do not confuse it with
`/api/documents/delete`, which removes indexed records.

## Use the import and OCR surfaces

Discover converted/import candidates through the schema and read endpoints.
Use the OCR scan endpoint to inspect which pages need OCR before queueing work.
Use fetch/convert/promote only after resolving exact files and permission tiers.

`POST /api/import/fetch` queues a fetch job whose `backend` is `auto` (the default),
`requests`, `crawl4ai`, `scrapling` or `crawlee`. `auto` tries crawl4ai, then
scrapling, then plain `requests`, using whatever is installed, and **never picks
`crawlee`**: that one is opt-in, by name. It renders the page with Crawlee's
`PlaywrightCrawler` in Chromium, then runs the same HTML-to-markdown conversion as
the others (the `format:"pdf"` print path does not use a backend). A backend that
is not installed, or whose crawl fails, makes the fetch job fail: read the job log,
report the cause, and let the operator choose another backend instead of retrying
with a different one yourself. In a container running as root, `crawlee` also needs
`CRAWLEE_DISABLE_BROWSER_SANDBOX=1` in its environment.

Inspect OCR readiness:

```cmd
curl.exe http://127.0.0.1:8052/api/ocr/status
```

The PaddleOCR lane has **three** states, not two, and collapsing them produces
the wrong advice every time:

| Reported | Meaning | Say |
|---|---|---|
| `configured: false` | No sidecar in the config at all. | It is not set up; offer to configure it. |
| `configured, reachable: false` | Nothing answers. | Almost always the container is deliberately stopped to reclaim memory between ingests. **Not a fault.** Offer to start it. |
| `reachable: true, engine_importable: false` | It answers HTTP and cannot OCR. | A real defect in the image. Quote `engine_error`; restarting will not help. |

`engine_importable: null` means unknown — an older sidecar image that predates
the field. Report it as unknown, never as broken: calling a working container
red is its own failure.

The sidecar returns **503** when its engine cannot import, so a bare
reachability test that treats any answer as "up" will file the third state
under the second.

The sidecar runs CPU or GPU on the same port and the same contract, so nothing
in the config distinguishes them. `device` on its `/health` is what tells you
which is running — and a GPU build that quietly fell back to CPU shows up only
as unexplained slowness, so read it before investigating page rates.

### Which ENGINE, which is a separate question from which device

The sidecar serves two, reported as `pipeline` (the default) and `pipelines`
(what this image can actually do):

| | `ocr` | `vl` |
|---|---|---|
| models | PP-OCRv6 detect + recognise | PP-DocLayoutV3 + PaddleOCR-VL-1.6-0.9B |
| output | plain text lines | markdown: headings, `$…$` LaTeX, HTML tables |
| cost | ~3 s/page, ~1.1 GB VRAM | ~30x that, ~7.9 GB VRAM |

Measured on a Pascal 8GB card over dense textbook pages: **`vl` is no better at
prose and roughly 30x slower**, so do not recommend switching a whole ingest to
it for a scanned novel. It wins where the STRUCTURE is the content — it
reconstructs a dynamic-programming table as real HTML with its headers, where
`ocr` returns the same digits with every relationship gone. That is a per-book
judgement (`pdf.paddle_ocr.pipeline`), not a global setting; on a 400-page book
the choice is roughly 20 minutes against 10 hours.

`pipelines.vl.available: false` always carries a `reason`. A CPU image has no
`PaddleOCRVL` at all and needs the GPU image; a GPU image built without the
`paddlex[ocr]` extras needs a rebuild. Quote the reason rather than guessing
which.

A request for an unavailable engine answers **501**. The sidecar never
substitutes the other one, so a page that came back is the engine that was
asked for.

Treat `POST /api/ocr/warm` as a mutating external action: confirm before sending
the tiny real model request because it may allocate or bill the configured
service. A successful warm-up proves that the OCR request path worked, not that
a later full ingest will succeed.

Choose `auto`, `tesseract`, `paddle`, `vlm`, or `none` only when currently
reported as valid. Start external OCR infrastructure through the project's
documented launcher when required; do not invent a model path or endpoint.

Avoid running memory-heavy OCR, evaluation, and query services together when
the host cannot support them.

## Retag indexed documents

Resolve exact source keys and preview representative chunks. Confirm the
metadata mutation, then call:

```cmd
curl.exe -X POST http://127.0.0.1:8052/api/documents/retag -H "Content-Type: application/json" -d "{\"source_files\":[\"<source-key>\"],\"add_tags\":[\"<tag>\"]}"
```

Use only schema-supported domain, course, add-tag, and remove-tag fields. Report
matched documents and updated chunks. Restart or rebuild only when the endpoint
or returned message says the warm/sparse state requires it.

## Delete indexed documents safely

Before deletion:

1. Query `/api/documents` and resolve exact source keys.
2. Read chunk counts and preview representative chunks.
3. Echo the exact deletion set.
4. Explain that vault/source files remain on disk.
5. Obtain explicit confirmation.

Then call:

```cmd
curl.exe -X POST http://127.0.0.1:8052/api/documents/delete -H "Content-Type: application/json" -d "{\"source_files\":[\"<source-key>\"]}"
```

Report dense, sparse, and JSONL effects returned by the endpoint. Stop on any
partial failure and surface it; do not hide it with cleanup or a broad rebuild.

## Switch or forget vault registrations

Read `/api/vaults` before switching. Treat `POST /api/vaults/switch` as a
mutation because it snapshots current settings and restores target settings.
Verify the target identity and paths returned by the API before confirming.

Treat forgetting a registration separately from deleting source data or indexed
documents. Follow the exact semantics reported by `/api/schema`.

## Restart and verify the warm query API

After an operation that changes indexes, loaded metadata, provider credentials,
or restart-required settings, confirm and call the management restart endpoint
reported by `/api/schema`, typically:

```cmd
curl.exe -X POST http://127.0.0.1:8052/api/service/restart
```

Poll `GET :8051/health` and read its `state`, not only `ready`:

| `state` | Meaning | Action |
|---|---|---|
| `loading` | Indexes and models still building. | Keep polling. Report progress. |
| `ready` | Warm. | Verify, then report success. |
| `failed` | The build raised. The process deliberately stays up to say why. | Read `error` and `hint`, then the startup log. Do not restart again blindly. |

A build failure is reported, never fatal: the service keeps answering `/health`
and returns 503 with the same reason from every retrieval endpoint, instead of
crash-looping under the supervisor while `/health` answers nothing at all.

When a restart comes back but never reports ready, tail the startup log rather
than guessing:

```cmd
curl.exe "http://127.0.0.1:8052/api/service/log?lines=120"
```

This is the other half of the restart lane. Quote the first causal error from
it; a restart that "did nothing" is almost always a build error recorded there.

Then verify with `GET :8051/config`, `/providers`, `/stats`, and a targeted
`/search`. Never claim that a restart or change worked until those checks pass.

## Triage failures explicitly

| Symptom | Action |
|---|---|
| Request rejected before a job ID | Fix schema validation; report that nothing ran. |
| Inbox conflict | Inspect duplicates; use force only after explicit confirmation. |
| Job failed | Read the log, preserve the first causal error, and retry only after correcting it. |
| `Reranking failed:` | Treat it as retrieval, inspect model/limit/device/HTTP settings, and preserve the underlying error. |
| `Reranking failed:` naming `retrieval.laya.enabled` | The experimental Laya reranker is off, or its package or checkpoint is missing. Offer the switch (see the experimental-features section) or let the caller pick another mode; never substitute one silently. |
| `Bad request:`, or an `error` field, on a retrieval call that answered HTTP 200 | A rejected per-call knob, not an outage. Fix the knob; restart nothing. |
| `/health` reports `failed` right after enabling the relevance gate | An enabled gate with no threshold, or an unknown scorer, is a startup error by design. Set the threshold or turn the gate off. |
| A `crawlee` fetch job fails | The package or its browser is missing, or, as root in a container, the sandbox flag is unset. It never falls back to another backend; report the cause from the job log. |
| `bench run` exits `2` or `1` | `2`: a mistake in the command, nothing ran. `1`: some question rows failed; read their `error` in `per_query.jsonl`, and report the run as partial. |
| `bench draft` exits `1` | Some seeds were skipped on LLM or transport errors; its summary counts them. Nothing written is lost. Run it again: those seeds are asked again. |
| `/health` reports `failed`: a collection "was embedded by … but the configured embedder is …" | The embedding fingerprint guard: `embedding.*` no longer describes the embedder that built the collection, and searching would compare vectors from two models. Set `embedding.*` back to the recorded one. Never reach for `rag stamp --force` to clear it: its sample check fails on a mismatched collection anyway, and `--force` exists only for a recorded fingerprint the operator confirms is wrong. |
| Startup log: a collection "has no embedding fingerprint" | It predates fingerprints and is served unchecked. With approval, run `rag stamp` once (`rag stamp --collection <name>` for the HyPE question collection): it re-embeds a sample of stored chunks and records the embedder only if they match their stored vectors. It writes a `<collection>.embedding.json` beside the store and changes no vector. |
| `bench run` refuses the test split | Working as designed. Do not add `--unseal-test` without the operator's explicit approval. |
| BGE base fails around long inputs | Ensure its configured maximum is no greater than 512. |
| HTTP reranker transport/status/schema error | Fix the external service; do not report a provider outage or silently fall back. |
| Provider key is present but incompatible | Replace it with the declared credential type; never bypass prefix checks. |
| MiniMax returns billing/auth failure | Confirm that the configured M3 Token Plan entry has a compatible `sk-cp-` subscription key and test reachability separately. |
| OCR readiness fails | Inspect `/api/ocr/status`, configured preset, and external OCR service before ingesting. |
| PaddleOCR sidecar reachable but every page fails | Read `engine_importable`/`engine_error`: the engine cannot import. A rebuild fixes it; a restart does not. |
| PaddleOCR sidecar unreachable | Usually stopped on purpose to reclaim memory. Offer to start it; do not report it as a fault. |
| Dense/sparse/JSONL counts diverge unexpectedly | Run read-side integrity checks before proposing a rebuild. |
| Query API shows stale state after a successful job | Restart `:8051`, poll health, and verify a targeted search. |
| Restart returns but `/health` never leaves `loading` | Tail `GET /api/service/log`; report the first causal error instead of restarting again. |
| `/health` reports `state:"failed"` | Read `error` and `hint`, then the startup log. The 503 from retrieval endpoints is the diagnosis, not a transport fault. |
| Every public reranker reports "repository not found" | Suspect a stale or revoked Hugging Face token, not a missing model. `POST /api/rerank/check` distinguishes 401 from genuinely absent. |

Raise errors explicitly. Never turn an unavailable dependency, partial write,
or failed restart into a success message.

## Keep the boundary with personal-rag

Use `personal-rag` to:

- Search and answer from the corpus.
- Inspect stable evidence.
- Compare presets, rerankers, and generation providers.
- Run iterative self-RAG over retrieved chunks.

Use this skill to:

- Change the corpus, metadata, settings, credentials, vault, or service state.
- Run the eval bench, review its question sets, and switch experimental features
  on or off.
- Monitor long-running operational work.
- Diagnose failures that require configuration or corpus action.

After each successful operation, hand verification back to `personal-rag` with
the smallest targeted read that proves the intended outcome.
