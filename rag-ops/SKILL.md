---
name: rag-ops
description: "Operate and diagnose a configured personal-RAG corpus through its management API: inspect documents and settings, ingest files, run OCR and indexing jobs, manage tags, remove indexed documents, switch vaults, store declared provider credentials, restart the warm query service, and monitor failures. Use for corpus or runtime operations, not for answering a question from the vault; use personal-rag for retrieval, cited answers, and query comparisons."
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

Validate reranker combinations before persisting:

- Keep `BAAI/bge-reranker-base` at or below its 512-token context limit.
- Use a model's declared hard context limit, not a limit copied from another
  model.
- Configure an HTTP URL/model before choosing `http` reranking.
- Treat invalid model/limit/device combinations as explicit failures.

Use `lexical` for a model-free low-power profile and `none` only for deliberate
fused-order diagnostics. Do not silently downgrade a failed configured
reranker.

## Queue and monitor jobs

Read `job_kinds` from `/api/schema`. Confirm mutating work, then enqueue one
specific job:

```cmd
curl.exe -X POST http://127.0.0.1:8052/api/jobs -H "Content-Type: application/json" -d "{\"kind\":\"<job-kind>\",\"params\":{}}"
```

Use the schema-defined parameters for common operations such as:

- PDF, notebook, code, Markdown, or URL ingest.
- Index append or full rebuild.
- BM25 rebuild.
- Scoped HyPE construction.
- Metadata recalibration.
- Retrieval evaluation.

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

Inspect OCR readiness:

```cmd
curl.exe http://127.0.0.1:8052/api/ocr/status
```

Treat `POST /api/ocr/warm` as a mutating external action: confirm before sending
the tiny real model request because it may allocate or bill the configured
service. A successful warm-up proves that the OCR request path worked, not that
a later full ingest will succeed.

Choose `auto`, `tesseract`, `vlm`, or `none` only when currently reported as
valid. Start external OCR infrastructure through the project's documented
launcher when required; do not invent a model path or endpoint.

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

Poll `GET :8051/health`, then verify with `GET :8051/config`, `/providers`,
`/stats`, and a targeted `/search`. Never claim that a restart or change worked
until those checks pass.

## Triage failures explicitly

| Symptom | Action |
|---|---|
| Request rejected before a job ID | Fix schema validation; report that nothing ran. |
| Inbox conflict | Inspect duplicates; use force only after explicit confirmation. |
| Job failed | Read the log, preserve the first causal error, and retry only after correcting it. |
| `Reranking failed:` | Treat it as retrieval, inspect model/limit/device/HTTP settings, and preserve the underlying error. |
| BGE base fails around long inputs | Ensure its configured maximum is no greater than 512. |
| HTTP reranker transport/status/schema error | Fix the external service; do not report a provider outage or silently fall back. |
| Provider key is present but incompatible | Replace it with the declared credential type; never bypass prefix checks. |
| MiniMax returns billing/auth failure | Confirm that the configured M3 Token Plan entry has a compatible `sk-cp-` subscription key and test reachability separately. |
| OCR readiness fails | Inspect `/api/ocr/status`, configured preset, and external OCR service before ingesting. |
| Dense/sparse/JSONL counts diverge unexpectedly | Run read-side integrity checks before proposing a rebuild. |
| Query API shows stale state after a successful job | Restart `:8051`, poll health, and verify a targeted search. |

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
- Monitor long-running operational work.
- Diagnose failures that require configuration or corpus action.

After each successful operation, hand verification back to `personal-rag` with
the smallest targeted read that proves the intended outcome.
