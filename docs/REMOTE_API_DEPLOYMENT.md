# Remote API deployment (llama.cpp-native)

This deployment provides a single authenticated endpoint for QMD remote mode using the same model runtime family (`llama.cpp`) as local execution.

Endpoints exposed by gateway:

- `POST /v1/embeddings`
- `POST /v1/chat/completions`
- `POST /v1/rerank` (proxied to backend `/rerank`)
- `POST /rerank`

## Topology

- `nginx` gateway (Bearer auth + routing)
- `llama-server` (embeddings)
- `llama-server` (generation)
- `llama-server` (reranking)

Compose files live in:

- `deploy/remote-api/docker-compose.yml` (CPU-safe base)
- `deploy/remote-api/docker-compose.gpu.yml` (optional NVIDIA override)
- `deploy/remote-api/nginx.conf.template`

## 1) Prepare model files

Put GGUF model files in a directory (example: `./models`) on the server.

Expected defaults:

- `embeddinggemma-300m-q8_0.gguf`
- `qmd-query-expansion-1.7b-q4_k_m.gguf`
- `qwen3-reranker-0.6b-q8_0.gguf`

## 2) Create `.env`

In `deploy/remote-api/` create:

```env
QMD_GATEWAY_API_KEY=replace-with-strong-token
QMD_MODEL_DIR=./models

QMD_EMBED_MODEL_FILE=embeddinggemma-300m-q8_0.gguf
QMD_GENERATE_MODEL_FILE=qmd-query-expansion-1.7b-q4_k_m.gguf
QMD_RERANK_MODEL_FILE=qwen3-reranker-0.6b-q8_0.gguf

# Optional runtime tuning
QMD_EMBED_CTX_SIZE=2048
QMD_GENERATE_CTX_SIZE=4096
QMD_RERANK_CTX_SIZE=2048
```

Why these context defaults:
- `embed=2048` is a practical low-memory setting for chunk-sized inputs.
- `rerank=2048` aligns with qmd-side rerank assumptions and keeps VRAM predictable.
- `generate=4096` gives query-expansion enough headroom without being too heavy.

You can lower them for tighter memory budgets or raise if your workload needs longer inputs.

## 3) Start stack

CPU mode (default):

```bash
cd deploy/remote-api
docker compose up -d
```

NVIDIA GPU mode:

```bash
cd deploy/remote-api
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up -d
```

## 4) Configure QMD client

```bash
export QMD_LLM_MODE=remote
export QMD_REMOTE_API_BASE_URL=http://<server>:8080
export QMD_REMOTE_API_KEY=<same-token>
export QMD_REMOTE_API_RERANK_PATHS=/v1/rerank,/rerank,/v1/re-rank
```

## 5) Smoke test

```bash
qmd embed
qmd query "test"
```

Success criteria:

- No local GGUF loading attempts on client machine
- Requests hit remote gateway
- Query still completes if rerank endpoint is unavailable (fallback)

## Multi-key auth (future)

You can add additional bearer keys without changing app code:

1. Create file(s) in `deploy/remote-api/authorized-keys/*.map` with entries:

```nginx
"Bearer sk-extra-1" 1;
"Bearer sk-extra-2" 1;
```

2. Restart gateway container.
