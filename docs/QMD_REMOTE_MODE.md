# QMD Remote LLM Mode (OpenAI-compatible APIs)

QMD now supports a **remote** mode for embeddings, query expansion generation, and reranking using OpenAI-compatible HTTP APIs.

Local mode remains the default.

## Mode selection

- `QMD_LLM_MODE=local` (default)
- `QMD_LLM_MODE=remote` (use remote API)

You can also enable remote mode by setting `QMD_REMOTE_API_BASE_URL`.

## Environment variables

### Required (remote mode)

- `QMD_REMOTE_API_BASE_URL`
  - Example: `http://gpu-host:8080`

### Optional (remote mode)

- `QMD_REMOTE_API_KEY`
  - Bearer token sent as `Authorization: Bearer <token>`
- `QMD_REMOTE_API_EMBED_MODEL`
  - Embedding model name passed to `/v1/embeddings`
- `QMD_REMOTE_API_GENERATE_MODEL`
  - Generation model name passed to `/v1/chat/completions`
- `QMD_REMOTE_API_RERANK_MODEL`
  - Rerank model name passed to rerank endpoint
- `QMD_REMOTE_API_TIMEOUT_MS`
  - Request timeout (default: `30000`)
- `QMD_REMOTE_API_RERANK_PATHS`
  - Comma-separated rerank endpoint candidates
  - Default: `/v1/rerank,/rerank,/v1/re-rank`

### Optional (local mode)

- `QMD_EMBED_MODEL`
- `QMD_GENERATE_MODEL`
- `QMD_RERANK_MODEL`

## Example: remote mode

```bash
export QMD_LLM_MODE=remote
export QMD_REMOTE_API_BASE_URL=http://gpu-host:8080
export QMD_REMOTE_API_KEY=replace-with-real-token

# Optional explicit models
export QMD_REMOTE_API_EMBED_MODEL=BAAI/bge-small-en-v1.5
export QMD_REMOTE_API_GENERATE_MODEL=Qwen/Qwen2.5-1.5B-Instruct
export QMD_REMOTE_API_RERANK_MODEL=BAAI/bge-reranker-base

# If your provider only supports /rerank, not /v1/rerank
export QMD_REMOTE_API_RERANK_PATHS=/v1/rerank,/rerank
```

Then run QMD as usual:

```bash
qmd index
qmd embed
qmd query "your query"
```

## Rerank endpoint compatibility & fallback

QMD tries rerank paths in order (default):

1. `/v1/rerank`
2. `/rerank`
3. `/v1/re-rank`

If all candidates are unsupported (404/405), QMD falls back to retrieval-order scoring so searches still complete.

## Migration notes

- Existing local installs need **no change**.
- To move to remote mode, set only:
  - `QMD_LLM_MODE=remote`
  - `QMD_REMOTE_API_BASE_URL=...`
- Keep the same CLI workflow (`qmd embed`, `qmd query`, MCP tools).
- In remote mode, tokenization/chunk sizing uses a byte-level fallback when local tokenizers are unavailable.
