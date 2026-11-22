---
name: deploy-bare-metal
description: Configure the env files for the whisper service (services/whisper) and deploy it on a host where the private-ai-stack root stack is already running. Use when asked to deploy, configure, set up, or bring up whisper on a server, or to check that whisper is deploy-ready.
---

# Deploy whisper on bare metal

Goal: `services/whisper` running on a host, with its env files filled in, reachable only on the
`private-ai-stack_backend` network as `http://whisper:8000/v1`, and verified from another container.

This service has no public hostname and no secrets. The only setting is `WHISPER__INFERENCE_DEVICE`.
Host setup (Docker Engine, the NVIDIA Container Toolkit, the firewall, the git clone) belongs to the root
skill, at `../../../../.claude/skills/deploy-bare-metal/SKILL.md` from this file's directory (the
root repo's `.claude/skills/deploy-bare-metal/SKILL.md`). Follow its Steps 1 to 3 first, then come back here.

## Ground rules

- **Confirm the target first.** Get a yes on the host and on any system change (packages, driver,
  runtime, `docker` group). Do not run `sudo` before that yes.
- **Do not read env files.** Never open `.env.whisper`, `.env.*`, or `.env` except the `*.example`
  templates. Write values with `set_val` (below), which prints nothing. Verify by counting matches, not
  by printing lines.
- **Do not change the stack's shape.** No `ports:`, no new services, no literal values in compose. This
  service stays in its own compose file, and the root compose files stay unchanged.
- **Do not start containers without approval.** Creating and validating files is fine. `./start.sh prod
  whisper` needs a yes first.
- **Do not download models without approval.** The first transcription downloads a Hugging Face model,
  which can be gigabytes.
- **Report faithfully.** Show the exact error for any failed check. Do not say the service works until
  the checks in Step 5 pass.

## Step 1: Preconditions (read-only)

```sh
cd services/whisper
git status --short                                   # service repo state
nvidia-smi --query-gpu=name,driver_version --format=csv,noheader   # GPU present?
docker compose version
docker network inspect private-ai-stack_backend --format '{{.Name}}'  # root stack must be up
docker volume inspect private-ai-stack_whisper_models --format '{{.Name}}' 2>&1  # cache, may not exist yet
```

Decide on each result:

- **No GPU** (`nvidia-smi` fails): stop. The compose file reserves an NVIDIA GPU, and the CUDA image
  will not start without it. Ask the user whether to use a CPU variant. A CPU variant needs a compose
  change (no `deploy` block, the CPU image tag) and `WHISPER__INFERENCE_DEVICE=cpu`. Make that change
  only with approval.
- **Backend network missing**: the root stack is not up. Run the root skill first. Do not create the
  network by hand.
- **Cache volume missing**: fine. Create it in Step 4, so the first start does not fail on the
  `external: true` volume.

## Step 2: Create `env/.env.whisper`

The template is `env/.env.whisper.example`, which the repo tracks. Copy it only if the real file does not
exist. Never overwrite an existing real file.

```sh
[ -f env/.env.whisper ] || cp env/.env.whisper.example env/.env.whisper
chmod 600 env/.env.whisper
```

The file must set exactly one key, `WHISPER__INFERENCE_DEVICE`, to the device the host can use:

| Host | Value |
|---|---|
| NVIDIA GPU present (Step 1 passed) | `cuda` |
| No GPU, after the user approved a CPU variant | `cpu` |

Write it with this helper. It prints nothing. It replaces the line if the key exists, and appends it if not:

```sh
set_val() {   # set_val <file> <KEY> <value>; prints nothing
  if grep -q "^$2=" "$1"; then sed -i "s|^$2=.*|$2=$3|" "$1"; else printf '%s=%s\n' "$2" "$3" >> "$1"; fi
}
set_val env/.env.whisper WHISPER__INFERENCE_DEVICE cuda
```

## Step 3: Check the env file (names and counts only)

Do not print the file. Check that the key is present exactly once, and that no other key is set:

```sh
grep -c '^WHISPER__INFERENCE_DEVICE=' env/.env.whisper     # expect 1
grep -c -v -E '^(#|$|WHISPER__INFERENCE_DEVICE=)' env/.env.whisper   # expect 0 (no stray keys)
grep -c -E '^WHISPER__INFERENCE_DEVICE=(cuda|cpu)$' env/.env.whisper # expect 1 (valid value)
```

Expect `1`, `0`, `1`. If the value check gives `0`, the value is not `cuda` or `cpu`. Fix it with
`set_val` and check again. Do not read the file to diagnose it.

The service has no API key, no password, and no `.env` link, so the compose file loads only
`./env/.env.whisper`. Nothing from the root `.env` is needed.

## Step 4: Validate the compose files (no containers started)

Run from the repo root, as `start.sh` does. The `--env-file` flag is for root `${VAR}` interpolation,
which this service does not use. `config -q` prints nothing, so no values are exposed:

```sh
cd /home/rie/Project/private-ai-stack
docker compose --env-file .env -f services/whisper/docker-compose.yml config -q && echo "prod OK"
docker compose --env-file .env -f services/whisper/docker-compose.yml -f services/whisper/docker-compose.dev.yml config -q && echo "dev OK"
```

Both must print OK. Then check the rules, all from compose files and names only:

```sh
grep -nE '^\s*ports:' services/whisper/docker-compose*.yml          # expect no output
grep -nE '^\s*environment:' services/whisper/docker-compose*.yml    # expect no output
```

If the cache volume is missing (Step 1), create it. Compose declares it `external`:

```sh
docker volume create private-ai-stack_whisper_models
```

## Step 5: Start and verify (needs approval)

```sh
./start.sh prod whisper
docker compose --env-file .env -f services/whisper/docker-compose.yml ps
docker compose --env-file .env -f services/whisper/docker-compose.yml logs --tail=100 whisper
```

Expect the container `running`. Startup logs show uvicorn listening on port 8000. A CUDA error at
startup means the GPU is not visible to the container. Check the NVIDIA runtime (root skill Step 3) and
stop there. Do not change the compose file to hide it.

Verify from another container on the backend network, using the agent-service container that already
exists. It is Node 18+, so `fetch` is available. Print only the HTTP status code:

```sh
docker compose -f docker-compose.yml exec -T agent-service node -e "fetch('http://whisper:8000/v1/models').then(r=>console.log(r.status)).catch(e=>{console.error(e.cause?.code||e.message);process.exit(1)})"
```

Expect `200`. Report the actual code. If the code is not `200`, look at the logs and do not claim success.
Only the status code is needed for this check. Do not print the response body.

Transcription test (only with approval, because it downloads a model). It uses the model from the
QUICKSTART, `Systran/faster-whisper-small`. Send a short sample file from inside the agent-service
container. Do not send user audio:

```sh
docker compose -f docker-compose.yml exec -T agent-service sh -c 'curl -s -o /dev/null -w "%{http_code}\n" -F "model=Systran/faster-whisper-small" -F "file=@/path/to/sample.wav" http://whisper:8000/v1/audio/transcriptions'
```

Expect `200`. If `curl` is not in the agent-service image, skip this and say so in the report.

## Step 6: Check the service repo

```sh
cd services/whisper
git status --short
git check-ignore -v env/.env.whisper     # must be ignored
git ls-files env/                        # must list only *.example (and env/.env.dev if it exists)
```

Stage and commit only when the user asks. Stage files by name. Never stage `env/.env.whisper`.

## Done when

- The env file has `WHISPER__INFERENCE_DEVICE` set once, to a valid value, and no other keys (Step 3).
- Both compose files pass `config -q`, and no `ports:` or `environment:` entries exist (Step 4).
- The container is `running` in `docker compose ps` (Step 5).
- The `/v1/models` check from the backend network returns `200` (Step 5).
- The user has been told what was done, what was not (for example, the transcription test if it was not
  approved), and what is left.

End with a short report: the device chosen, the checks that passed with their codes, the checks that were
skipped and why, and any step that still needs the user.
