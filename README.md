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
| `CTX_SIZE` | 65536 | Context length, shared by input and output |
| `N_GPU_LAYERS` | auto | `auto` lets `--fit` split layers and MoE experts between GPU and RAM to fit `CTX_SIZE`; a number disables fitting |
| `FIT_TARGET` | 256 | VRAM (MiB) `--fit` leaves free for the desktop and other apps |
| `CACHE_TYPE_K` / `CACHE_TYPE_V` | q8_0 | KV cache type; `q4_0` halves context memory. Keep both the same, mixed types fall back to CPU |
| `BATCH_SIZE` / `UBATCH_SIZE` | 2048 / 512 | Prompt batch sizes; `UBATCH_SIZE=2048` speeds up prompts for MoE models with experts in RAM |
| `THREADS` | 4 | CPU threads; match physical cores |
| `EXTRA_ARGS` | | Extra `llama-server` flags, e.g. `--n-cpu-moe 32` |

`.env.example` holds commented presets for each tested model.
The first request after a start is slow while weights page into memory.

## Endpoints

- Web UI: `http://localhost:11435/`
- Health: `http://localhost:11435/health`
- Models: `http://localhost:11435/v1/models`

## Tested models (RTX 2060 Super, 8 GB)

| Model | Size | `.env` settings | Speed (gen / prompt) |
|---|---|---|---|
| [Ornith-1.5-9B-Q4_K_M](https://huggingface.co/ornith-ai/Ornith-1.5-9B-GGUF) | 5.8 GB | defaults (64k, q8_0) | 52 / 1260 tok/s, fully on GPU |
| | | `CTX_SIZE=131072`, q4_0 | 23 / 1020 tok/s |
| | | `CTX_SIZE=262144`, q4_0 | 9 / 690 tok/s |
| [Ornith-1.5-9B-uncensored.Q4_K_M](https://huggingface.co/mradermacher/Ornith-1.5-9B-uncensored-GGUF) | 5.6 GB | as above | Same architecture as above |
| [Qwen3-Coder-30B-A3B-Instruct-UD-Q3_K_XL](https://huggingface.co/unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF) | 13.8 GB | `CTX_SIZE=131072`, q4_0, `UBATCH_SIZE=2048` | 14 / 640 tok/s; experts mostly in RAM |
| | | `CTX_SIZE=262144`, q4_0, `UBATCH_SIZE=2048` | 7 / 560 tok/s; model's full native context |
| [ornith-1.0-9b-Q4_K_M](https://huggingface.co/deepreinforce-ai/Ornith-1.0-9B-GGUF) | 5.6 GB | defaults | |
| [Qwen3-4B-Instruct-2507-Q8_0](https://huggingface.co/unsloth/Qwen3-4B-Instruct-2507-GGUF) | 4.3 GB | `CTX_SIZE=32768` | |
| [Qwen2.5-Coder-7B-Instruct-Q6_K](https://huggingface.co/bartowski/Qwen2.5-Coder-7B-Instruct-GGUF) | 6.3 GB | `CTX_SIZE=32768` | Tool calls come back as raw text; chat only, not opencode |
| Qwen3.6-27B-Q4_K_M | 16 GB | `CTX_SIZE=32768`, `N_GPU_LAYERS=20` | ~0.3 tok/s on an i7-3770; full offload hangs the machine |

Speeds are warm, with a 4k-token prompt, on an i7-3770 with ~0.8 GB of VRAM used by the desktop.

## opencode

`opencode.json` defines a `llamacpp` provider for the Ornith 1.5 and Qwen3-Coder models. Copy it to your opencode config or project root.
Its `baseURL` port and `limit.context` must match `LLAMA_PORT` and `CTX_SIZE` in `.env`.
