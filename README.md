## Setup

```bash
git clone https://github.com/carlosesteven/MI-API.git
cd MI-API
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
uvicorn api:app --host 0.0.0.0 --port 8000
```

Then open `http://localhost:8000/` for interactive API docs.

<br>

## Disclaimer

This project is for educational purposes and API integrity research only. The author takes absolutely zero responsibility for network usage. Code contains zero skiddable artifacts.
