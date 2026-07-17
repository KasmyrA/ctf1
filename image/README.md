# CTF Library Assistant — self-hosted migration plan

## Target architecture
```
browser --> nginx (frontend, :8080) --/api/*--> backend (Express, :8787) --> ollama (:11434)
                                                                                 |-- chat model
                                                                                 |-- guardian model
```
Only `frontend` is published on the host. `backend` and `ollama` stay on the
internal Docker network — the browser never talks to Ollama directly, and
CORS is no longer needed because everything is same-origin behind nginx.

## Files in this stack
- `docker-compose.yml` — three services: `ollama`, `backend`, `frontend`.
- `backend/server.js` — Express replacement for `worker.js`. Same request/
  response shape (`POST {prompt} -> {response}` / `{error}`), same system
  prompt logic, same 4000-char limit.
- `backend/Dockerfile`, `backend/package.json`
- `frontend/html/index.html` — same UI, just points at `/api/chat` (relative)
  instead of the `workers.dev` URL.
- `frontend/nginx.conf` — serves the static page and reverse-proxies `/api/`.

## Model choice — important correction
`granite4.1-guardian:8b` is **not** a conversational model — it's IBM's
safety/judge model: it only scores whether a prompt or response meets a
given risk criterion (`yes`/`no`, optionally with a `<think>` trace). It
can't replace Gemini as the thing that talks to the user.

So the plan uses two models:
1. **Chat model** (`CHAT_MODEL`, default `granite3.1-dense:8b`) — generates
   the actual reply. Any solid instruct model works here (`llama3.1:8b`,
   `qwen2.5:7b`, `gemma2:9b`, `granite4.1:8b`...). With 128GB RAM you have
   headroom to try several and pick by feel.
2. **Guard model** (`GUARD_MODEL`, default `granite4.1-guardian:8b`) —
   runs *after* the chat model, given the user prompt + generated reply,
   with a custom criterion: "does this reply disclose the flag". If it
   scores `yes`, the backend withholds the answer instead of returning it.
   This is a second line of defense against prompt-injection bypasses on
   top of the system prompt itself — arguably more interesting for a CTF
   challenge than the Gemini version had, since now there's a guard layer
   to also try to bypass. `server.js` fails **closed**: if the guard call
   errors, the reply is withheld rather than risk leaking the flag.

The Granite Guardian chat template expects the criteria appended as a
trailing `<guardian>...</guardian>` user turn and returns
`<score>yes|no</score>`; `server.js` already builds that turn and parses
the score. Test this against the real model before relying on it — BYOC
(bring-your-own-criteria) behavior should be verified, per IBM's own model
card, before production use.

## Setup steps

1. **Build and start the stack**
   ```bash
   cd ctf-stack
   docker compose up -d --build
   ```

2. **Pull the models into the Ollama container** (first run only; they
   persist in the `ollama_data` volume after this)
   ```bash
   docker exec -it ollama ollama pull granite3.1-dense:8b
   docker exec -it ollama ollama pull granite4.1-guardian:8b
   ```

3. **Set the real flag** — don't leave it in git. Either put it in a
   `.env` file next to `docker-compose.yml`:
   ```
   SECRET_FLAG=FLAG{your_real_flag_here}
   ```
   and reference `${SECRET_FLAG}` in the compose file's `environment:`
   block, or inject it via your orchestrator's secret store at deploy time.

4. **AMD GPU (Ryzen AI 395) acceleration** — `ollama/ordinaryllama:rocm`
   needs `/dev/kfd` and `/dev/dri` passed through (already in the compose
   file) and the host's `render` group permissions for the user running
   Docker. Confirm GPU is picked up:
   ```bash
   docker exec ollama ollama ps
   ```
   If ROCm isn't cooperating, switch the image tag to `ollama/ollama:latest`
   for a CPU fallback while you debug.

5. **Test end to end**
   ```bash
   curl -s http://localhost:8080/api/chat \
     -H 'Content-Type: application/json' \
     -d '{"prompt":"Poleć mi książkę o kryptografii"}' | jq
   ```

6. **Open the UI** at `http://localhost:8080`.

## Other differences from the Worker version
- Rate limiting: the Worker had none; `server.js` adds a simple in-memory
  per-IP limiter (20 req/min). Swap the `Map` for Redis if you scale to
  multiple backend replicas.
- Logging: same `console.error` pattern, visible via `docker compose logs
  backend`.
- `MAX_PROMPT_LENGTH` and both model names are now env vars, not constants,
  so you can retune without rebuilding the image.
- If you want to keep the D1/SQLite-based analytics from your other CTF
  worker, add a fourth service (e.g. `litestream`+SQLite volume, or
  Postgres) and log each request/response pair from `server.js` — happy to
  sketch that out separately if you want it in this same plan.
