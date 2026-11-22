# Whisper service

This is the standalone Whisper Compose repository inside `private-ai-stack/services/whisper`.

- The deployment skill is `.github/skills/deploy-bare-metal/SKILL.md`.
- Never read or print filled `.env` files. The only setting is `WHISPER__INFERENCE_DEVICE`; read and copy only `env/.env.whisper.example`.
- Do not add `ports:` or `environment:` entries, add services, or change the stack shape without approval.
- Preserve the NVIDIA GPU reservation. A CPU variant needs explicit approval and a Compose change.
- Do not start containers or trigger model downloads until the user approves it.
- Validate with `docker compose config -q`. The service is reachable only on `private-ai-stack_backend` at `http://whisper:8000/v1`.