# whisper

> This directory is a sub-project of [`private-ai-stack`](https://github.com/HoHuiHsieh/private-ai-stack).

Speaches (faster-whisper), an OpenAI-compatible speech-to-text server. Internal only: no host port.

- Endpoint on the backend network: `http://whisper:8000/v1` (`POST /v1/audio/transcriptions`).
- No API key. Models are chosen per request (`model` field) and downloaded from Hugging Face on first use.
- Image: `ghcr.io/speaches-ai/speaches:latest-cuda` (NVIDIA GPU).

## Run

```
./start.sh prod whisper     # from the repo root; the root stack must be up first
```

Logs:

```
docker compose --env-file .env -f services/whisper/docker-compose.yml logs -f whisper
```

## Files

- `docker-compose.yml`: prod service. Models are cached in the `private-ai-stack_whisper_models` volume.
- `docker-compose.dev.yml`: empty override, so `./start.sh dev whisper` works.
- `env/.env.whisper`: real settings (gitignored). `env/.env.whisper.example` is the tracked template.

The model cache volume is declared `external`, so models are kept across restarts.
