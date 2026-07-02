# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Lectura obligatoria al iniciar sesión

Lee [`SESSION_LOG.md`](./SESSION_LOG.md) antes de cualquier tarea. Contiene el historial de cambios realizados en sesiones anteriores: qué se implementó, qué archivos se modificaron y decisiones de diseño tomadas. Actualiza ese archivo al final de cada sesión con un resumen de lo que hiciste.

## Commands

```bash
# Install dependencies
pip install -r requirements.txt

# Run locally
uvicorn api:app --host 0.0.0.0 --port 8000 --reload

# Docker
docker build -t miruro-api .
docker run -p 8000:8000 miruro-api
```

There is no test suite and no linting configuration.

## Architecture

The entire API lives in a single file: `api.py` (~1100 lines). It is a FastAPI app that acts as a thin, authenticated proxy over two upstream sources:

1. **AniList GraphQL** (`https://graphql.anilist.co`) — all anime metadata: search, filter, collections, info, characters, relations, recommendations.
2. **Miruro Pipe** (`{MIRURO_BASE_URL}/api/secure/pipe`) — episode lists and M3U8 streaming URLs. Miruro's pipe protocol base64-encodes and gzip-compresses every request and response; `_encode_pipe_request()` and `_decode_pipe_response()` handle this transparently. The pipe sits behind Cloudflare TLS-fingerprint bot detection, so requests go through `curl_cffi` (`impersonate="chrome110"`) instead of `httpx` — a global `pipe_session` is reused across requests and auto-replaced on failure (see `_fetch_raw_episodes`, `_fetch_raw_recents`, `get_sources`).

### Security middleware (`secure_api`)

Every non-doc request must pass one of two checks (checked in order):
- Valid `x-api-key` header matching `API_KEY` env var
- `Origin` or `Referer` header that starts with one of the `ALLOWED_ORIGINS`

Doc paths (`/`, `/docs`, `/redoc`, `/openapi.json`) bypass this check entirely.

### ID encoding

Episode IDs returned by the Miruro pipe are base64-encoded. `_translate_id()` decodes a single ID; `_deep_translate()` recursively walks any JSON structure and decodes all IDs. Endpoints that return episode data must call `_deep_translate()` before returning.

### Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `ALLOWED_ORIGINS` | localhost variants | Comma-separated CORS + auth whitelist |
| `API_KEY` | `123456` | Auth header value (`x-api-key`) |
| `API_DEBUG` | `False` | `True` renders a styled HTML homepage; `False` renders a minimal page |
| `REDIS_HOST` | `localhost` | Redis host for the `/recent-episodes` cache |
| `REDIS_PORT` | `6379` | Redis port |
| `REDIS_PASSWORD` | — | Redis password |
| `CACHE_RECENT_EPISODES_HOURS` | `2` | TTL (hours) for the `/recent-episodes` cache |
| `MIRURO_BASE_URL` | — (required) | Base domain for the Miruro pipe, e.g. `https://www.miruro.to`. No hardcoded fallback — update this if Miruro changes domains again |
| `PIPE_USER_AGENT` | `Mozilla/5.0 (Windows NT 10.0; Win64; x64)` | User-Agent sent to the pipe |
| `PIPE_EXTRA_HEADERS` | `{}` | JSON object merged into pipe request headers (e.g. `sec-ch-ua`, `accept`, `cf_clearance`-adjacent headers) — used to adapt to Cloudflare without touching code |

### Deployment targets

- **Vercel**: `vercel.json` maps all routes to `api.py` via the Python runtime (`mangum` adapter is imported for ASGI compatibility).
- **Koyeb/Docker**: `Dockerfile` uses Python 3.11 slim, installs requirements, and starts uvicorn directly.
