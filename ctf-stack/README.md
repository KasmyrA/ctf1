# CTF Library Assistant — self-hosted stack

## Target architecture
```
browser --> nginx (frontend, :8080) --/api/*--> backend (Express, :8787) --> ollama (:11434)
```
Only `frontend` is published on the host. `backend` and `ollama` stay on the
internal Docker network — the browser never talks to Ollama directly, and
CORS is no longer needed because everything is same-origin behind nginx.

## Files in this stack
- `docker-compose.yml` — three services: `ollama`, `backend`, `frontend`.
- `server.js` — Express backend. It accepts `POST /api/chat` with `{prompt}`
   and returns `{response}` or `{error}`. The backend keeps the system prompt,
   the 4000-character prompt limit, and a simple in-memory per-IP rate limiter.
- `Dockerfile` and `package.json` — backend image and dependencies.
- `index.html` — static UI served by nginx.
- `nginx.conf` — serves the page and reverse-proxies `/api/` to the backend.

## Model choice
The backend uses a single chat model, configured with `CHAT_MODEL` and
defaulting to `gemma4:e2b` in `docker-compose.yml`. Any instruct/chat model
available in Ollama should work as long as it can answer in plain text.

The flag itself is injected through `SECRET_FLAG` in the compose file instead
of being hard-coded in the source. That keeps the challenge easy to tune while
still leaving the secret visible to the model through the system prompt.

## Setup steps

1. **Build and start the stack**
   ```bash
   cd ctf-stack
   docker compose up -d --build
   ```


2. **Set the real flag** — don't leave it in git. Replace the example value in
   the `SECRET_FLAG` entry inside `docker-compose.yml`, or inject it from your
   orchestrator's secret store at deploy time.

3. **Set the right acceleration for LLM** — `ollama/ollama:rocm`
   needs `/dev/kfd` and `/dev/dri` passed through (already in the compose
   file) and the host's `render` group permissions for the user running
   Docker. Confirm GPU is picked up:
   ```bash
   docker exec ollama ollama ps
   ```
   If ROCm isn't cooperating, switch the image tag in `docker-compose.yml` to
   `ollama/ollama:latest` for a CPU fallback while you debug.

4. **Test end to end**
   ```bash
   curl -s http://localhost:8080/api/chat \
     -H 'Content-Type: application/json' \
     -d '{"prompt":"Poleć mi książkę o kryptografii"}' | jq
   ```

5. **Open the UI** at `http://localhost:8080`.

## Notes
- Rate limiting is intentionally simple and in-memory, so it resets when the
   backend restarts and does not scale across replicas.
- `MAX_PROMPT_LENGTH` and `CHAT_MODEL` are environment variables, so you can
   tune the stack without rebuilding the image.
- Backend errors are logged with `console.error` and are visible through
   `docker compose logs backend`.
