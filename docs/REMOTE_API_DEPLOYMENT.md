# Remote API Deployment (low-VRAM GPU host)

This runbook deploys a single endpoint for QMD-compatible remote inference:

- `POST /v1/embeddings`
- `POST /v1/chat/completions`
- `POST /v1/rerank` (gateway-adapted to backend `/rerank`)

It uses:

- **Nginx gateway** for auth + routing
- **vLLM (embed)** for embeddings
- **vLLM (generate)** for query expansion generation
- **Infinity** for reranking

## 1) Prerequisites

- Linux GPU host with NVIDIA runtime
- Docker + Docker Compose plugin
- 6–12 GB VRAM minimum (8 GB recommended)

## 2) Configure environment

Create `.env` next to `docker-compose.yml`:

```env
QMD_GATEWAY_API_KEY=replace-with-strong-token

# Tune for your GPU budget
QMD_EMBED_GPU_UTIL=0.25
QMD_GENERATE_GPU_UTIL=0.45

# Model choices (small defaults for constrained VRAM)
QMD_EMBED_MODEL=BAAI/bge-small-en-v1.5
QMD_GENERATE_MODEL=Qwen/Qwen2.5-1.5B-Instruct
QMD_RERANK_MODEL=BAAI/bge-reranker-base
```

## 3) Start services

```bash
docker compose pull
docker compose up -d
```

## 4) Validate endpoints

```bash
# Embeddings
curl -s http://localhost:8080/v1/embeddings \
  -H "Authorization: Bearer $QMD_GATEWAY_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"BAAI/bge-small-en-v1.5","input":"hello world"}'

# Chat completions
curl -s http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer $QMD_GATEWAY_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen2.5-1.5B-Instruct","messages":[{"role":"user","content":"say hi"}]}'

# Rerank
curl -s http://localhost:8080/v1/rerank \
  -H "Authorization: Bearer $QMD_GATEWAY_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"BAAI/bge-reranker-base","query":"api auth","documents":["auth token guide","css layout tips"]}'
```

## 5) Point QMD to the remote API

```bash
export QMD_LLM_MODE=remote
export QMD_REMOTE_API_BASE_URL=http://localhost:8080
export QMD_REMOTE_API_KEY=$QMD_GATEWAY_API_KEY

# Optional: if your rerank backend is custom
export QMD_REMOTE_API_RERANK_PATHS=/v1/rerank,/rerank
```

## Operational notes

- If VRAM is tight, reduce `QMD_GENERATE_GPU_UTIL` first.
- If rerank endpoint is missing, QMD will fall back to retrieval-order scoring.
- For production, add TLS at the gateway (or terminate TLS upstream).
- Restrict network exposure (firewall + allowlist callers).
