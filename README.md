# TDS Project — LLM Task Runner

A FastAPI + Gradio service that auto-generates full-stack web projects using an LLM, pushes them to GitHub, and deploys via GitHub Pages.

## Stack

- **FastAPI** — REST API server
- **Gradio** — minimal UI mounted at `/ui`
- **httpx** — async HTTP client for LLM and GitHub API calls
- **aiofiles** — async file I/O
- **Docker** — containerized deployment (Python 3.11-slim)

## Project Structure

```
main.py          # Core app: FastAPI routes, LLM generation, GitHub integration
evaluation.py    # Example client script to submit a Round 1 task
requirements.txt # Python dependencies
Dockerfile       # Container build config
```

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | Status and available endpoints |
| GET | `/health` | Config health check (token/key presence) |
| POST | `/handle_request` | Main entry point — triggers Round 1 or Round 2 |
| GET | `/ui` | Gradio status UI |
| GET | `/docs` | Auto-generated Swagger docs |

## How It Works

### Round 1 — Initial Build
1. Receives a task brief, evaluation checks, and optional attachments.
2. Calls the LLM (`gpt-4o` via `OPENAI_BASE_URL`) to generate a full project as a JSON array of `{filename, content}` objects.
3. Saves files to `/tmp/{task_name}/`, including any attachments (base64, URL, or raw text).
4. Creates a public GitHub repo and pushes all files via the GitHub Contents API.
5. Enables GitHub Pages and polls until the site is live.
6. POSTs the evaluation payload (repo URL, commit SHA, pages URL) to `evaluation_url`.

### Round 2 — Revision
1. Clones the existing repo for the task.
2. Reads all existing files and passes them as context to the LLM.
3. LLM returns only new/updated files; these are written back and committed.
4. Force-pushes to GitHub, re-deploys Pages, and submits evaluation.

## Request Payload (`POST /handle_request`)

```json
{
  "secret": "<your_secret>",
  "email": "user@example.com",
  "task": "repo-name",
  "round": 1,
  "nonce": "unique-nonce-1234",
  "brief": "Build a Bootstrap page that...",
  "evaluation_url": "https://your-webhook-url",
  "checks": [
    {"js": "document.querySelector('#some-id').tagName === 'FORM'"}
  ],
  "attachments": [
    {"name": "data.csv", "url": "https://..."}
  ]
}
```

Supported attachment types: images (png/jpg/gif/svg), csv, json, audio (mp3/wav/ogg/flac/m4a), html, css, js, and plain text.

## Environment Variables

| Variable | Description |
|----------|-------------|
| `GITHUB_TOKEN` | GitHub personal access token |
| `OPEN_API_KEY` | LLM API key |
| `OPENAI_BASE_URL` | LLM endpoint (default: `https://aipipe.org/openrouter/v1/chat/completions`) |
| `secret` | Shared secret to authenticate incoming requests |

## Running Locally

```bash
pip install -r requirements.txt
GITHUB_TOKEN=... OPEN_API_KEY=... secret=... uvicorn main:app --host 0.0.0.0 --port 7860
```

## Docker

```bash
docker build -t tds-project .
docker run -p 7860:7860 \
  -e GITHUB_TOKEN=... \
  -e OPEN_API_KEY=... \
  -e secret=... \
  tds-project
```

## Example Client (`evaluation.py`)

Sends a Round 1 request to generate a Bootstrap page that fetches a GitHub username and displays the account creation date:

```bash
python evaluation.py
```

Update `HF_API_URL`, `email`, and `secret` in the script before running.

## License

MIT
