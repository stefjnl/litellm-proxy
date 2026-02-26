# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Standalone LiteLLM proxy service running as a Docker container. Acts as a unified OpenAI-compatible API gateway that routes requests to various LLM providers via NanoGPT. Used as shared infrastructure for local development across multiple Next.js apps.

## Architecture

- **No application code** — this is a config-only repo (Docker Compose + LiteLLM YAML config + env vars)
- **LiteLLM image**: `docker.litellm.ai/berriai/litellm:main-latest` (pulled from LiteLLM's registry)
- **Proxy listens on port 4000**, exposed to host
- **Two upstream routes**:
  - **NanoGPT** (`https://nano-gpt.com/api/v1`) — cloud models using `openai/` provider prefix, authenticated with `NANOGPT_API_KEY`
  - **Local** (`http://100.74.43.85:8080/v1`) — self-hosted model on Azerty via Tailscale, no API key
- **Auth**: Clients authenticate with `LITELLM_MASTER_KEY`
- **Default model**: `qwen3.5-local` — routes to local Qwen 3.5 35B-A3B instance
- **Fallback**: `qwen3.5-local` → `qwen3.5` (cloud) — if local is unreachable, automatically falls back to NanoGPT
- **No database** — runs stateless with `allow_requests_on_db_unavailable: true`

## Commands

```bash
# Start the proxy
docker compose up -d

# View logs
docker compose logs -f litellm

# Restart after config changes
docker compose restart litellm

# Health check
curl http://localhost:4000/health

# Test a model
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-local-dev-key" \
  -H "Content-Type: application/json" \
  -d '{"model": "qwen3.5-local", "messages": [{"role": "user", "content": "hello"}]}'
```

## Current Models

Local: `qwen3.5-local` (default) — Qwen 3.5 35B-A3B on Azerty via Tailscale
Cloud (NanoGPT): `qwen3.5` (cloud fallback), `glm-5`, `minimax-m2.5`, `kimi-k2.5`
Embedding: `qwen3-embedding`

## Adding a New Model

Add an entry to `litellm_config.yaml` under `model_list`:

```yaml
- model_name: <friendly-name>        # Name clients use
  litellm_params:
    model: openai/<provider>/<model>  # NanoGPT model path
    api_base: https://nano-gpt.com/api/v1
    api_key: os.environ/NANOGPT_API_KEY
```

Then restart: `docker compose restart litellm`

## Key Config

- `litellm_config.yaml` — model routing, LiteLLM settings
- `docker-compose.yml` — container config (image, ports, volumes, healthcheck)
- `.env` — secrets (master key, salt key, NanoGPT API key)
- `drop_params: true` — silently drops unsupported params instead of erroring
