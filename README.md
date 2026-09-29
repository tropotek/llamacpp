# llamacpp

Local llama.cpp `llama-server` in Docker with NVIDIA GPU offload, serving an OpenAI-compatible API on `http://localhost:11435/v1`.

## Requirements

- Docker with the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
- ~10 GB for the image plus the size of your GGUF models

## Setup

1. Put GGUF files in `models/` (gitignored).
2. `cp .env.example .env` and set `MODEL` to a file name in `models/`.
3. `docker compose up -d`

The container shares port 11435 with ollama1's llama-cpp service; run `docker compose -f docker-compose-llama.yml down` in ollama1 first.

## Switching models

Edit `MODEL` (and tuning vars) in `.env`, then `docker compose up -d --force-recreate`.
The model id reported by `/v1/models` is the file name.

| Variable | Default | Purpose |
|---|---|---|
| `LLAMA_PORT` | 11435 | Host port |
| `MODEL` | Ornith-1.5-9B-Q4_K_M.gguf | GGUF file in `models/` |
| `CTX_SIZE` | 65536 | Context length |
| `N_GPU_LAYERS` | 99 | Layers offloaded to GPU (99 = all) |
| `N_CPU_MOE` | 0 | MoE layers whose experts stay in system RAM |
| `THREADS` | 8 | CPU threads |

## Endpoints

- Web UI: `http://localhost:11435/`
- Health: `http://localhost:11435/health`
- Models: `http://localhost:11435/v1/models`

## Tested models (RTX 2060 Super, 8 GB)

| Model | Size | `.env` settings | Notes |
|---|---|---|---|
| [Ornith-1.5-9B-Q4_K_M](https://huggingface.co/deepreinforce-ai) | 5.8 GB | defaults | Hybrid attention/SSM; only 8 of 32 layers keep KV cache, so 64k ctx is cheap. ~7.2 GB VRAM used |
| Ornith-1.5-9B-uncensored.Q4_K_M | 5.6 GB | defaults | Same architecture as above |
| [ornith-1.0-9b-Q4_K_M](https://huggingface.co/deepreinforce-ai/Ornith-1.0-9B-GGUF) | 5.6 GB | defaults | ~1 GB VRAM headroom left |
| [Qwen3-4B-Instruct-2507-Q8_0](https://huggingface.co/unsloth/Qwen3-4B-Instruct-2507-GGUF) | 4.3 GB | `CTX_SIZE=32768` | ~7.5 GB VRAM used |
| [Qwen2.5-Coder-7B-Instruct-Q6_K](https://huggingface.co/bartowski/Qwen2.5-Coder-7B-Instruct-GGUF) | 6.3 GB | `CTX_SIZE=32768` | Tool calls come back as raw text; chat only, not opencode |
| [Qwen3-Coder-30B-A3B-Instruct-UD-Q3_K_XL](https://huggingface.co/unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF) | 13.8 GB | `CTX_SIZE=32768`, `N_CPU_MOE=32` | ~10 tok/s, RAM-bandwidth bound; raise `N_CPU_MOE` to free VRAM |
| Qwen3.6-27B-Q4_K_M | 16 GB | `CTX_SIZE=32768`, `N_GPU_LAYERS=20` | ~0.3 tok/s on an i7-3770; full offload hangs the machine |

## opencode

`opencode.json` defines a `llamacpp` provider for both Ornith 1.5 models. Copy it to your opencode config or project root.
Its `baseURL` port and `limit.context` must match `LLAMA_PORT` and `CTX_SIZE` in `.env`.
