# llamacpp

- Start: `docker compose up -d`; stop: `docker compose down`; logs: `docker logs -f llama-cpp`
- Model/tuning: `.env` (see `.env.example`); recreate with `docker compose up -d --force-recreate`
- `models/` is gitignored; never commit GGUF files.
