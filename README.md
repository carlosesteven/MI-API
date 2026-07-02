# MI API

FastAPI proxy over AniList GraphQL and the MI pipe. Returns anime metadata, episode lists, and streaming sources.

---

## Setup

```bash
git clone https://github.com/carlosesteven/MI-API.git
cd MI-API
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Environment variables

Create a `.env` file in the project root:

```env
ALLOWED_ORIGINS=http://localhost:3000,https://yourdomain.com
API_KEY=your_secret_key
API_DEBUG=False

# Redis cache (optional — API works without it)
REDIS_HOST=your_redis_host
REDIS_PORT=6379
REDIS_PASSWORD=your_redis_password
CACHE_RECENT_EPISODES_HOURS=2
```

| Variable                      | Default     | Purpose                               |
| ----------------------------- | ----------- | ------------------------------------- |
| `ALLOWED_ORIGINS`             | —           | Comma-separated CORS + auth whitelist |
| `API_KEY`                     | —           | Auth header value (`x-api-key`)       |
| `API_DEBUG`                   | `False`     | `True` renders full HTML docs at `/`  |
| `REDIS_HOST`                  | `localhost` | Redis host for caching                |
| `REDIS_PORT`                  | `6379`      | Redis port                            |
| `REDIS_PASSWORD`              | —           | Redis password                        |
| `CACHE_RECENT_EPISODES_HOURS` | `2`         | TTL for `/recent-episodes` cache      |

### Run locally

```bash
python -m uvicorn api:app --host 0.0.0.0 --port 8000 --reload
```

Open `http://localhost:8000/` for interactive API docs (requires `API_DEBUG=True`).

---

## Deploy (production — uvicorn direct)

### First deploy

```bash
git clone https://github.com/carlosesteven/MI-API.git
cd MI-API
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
nohup python -m uvicorn api:app --host 0.0.0.0 --port 8848 > /dev/null 2>&1 &
```

### Update to latest version

```bash
# 1. Pull changes
git pull

# 2. Install any new dependencies
pip install -r requirements.txt

# 3. Find the exact PID of this service (do NOT kill others)
ps aux | grep uvicorn

# 4. Kill only this process by PID
kill <PID>

# 5. Start again
nohup python -m uvicorn api:app --host 0.0.0.0 --port 8848 > /dev/null 2>&1 &
```

> **Important:** always kill by PID (`kill <PID>`), not by name (`pkill`). The server may be running multiple uvicorn processes on different ports.

## Disclaimer

This project is for educational purposes and API integrity research only. The author takes absolutely zero responsibility for network usage. Code contains zero skiddable artifacts.
