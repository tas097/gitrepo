# API Cheat Sheet — FastAPI & serving

---

## Batch job vs service

```
  BATCH JOB                        SERVICE
  start → work → print → EXIT      start → wait ──┐
                                        ▲         │
  nothing can ASK it anything           │   request arrives
                                        │         │
                                        │     respond
                                        └─────────┘
                                        keeps waiting
```

**You don't deploy a script; you deploy something that stays up.**

**The visible tell:** every command before this printed and gave your prompt back. `uvicorn` starts and **never returns the prompt**. That hanging terminal *is* the service.

→ Use a **second terminal** for everything else.

---

## Vocabulary

| Term | Means |
|---|---|
| **HTTP** | The protocol. A **request** goes out, a **response** comes back |
| **Endpoint** | One address that does one thing — `/health`, `/totals` |
| **Method** | The verb — `GET` (fetch), `POST` (send data in) |
| **JSON** | Text format for structured data. Looks like a Python dict |
| **Port** | A numbered door. `8000` by convention in development |
| **localhost / 127.0.0.1** | This machine talking to itself |
| **Header** | Extra metadata on a request — `accept: application/json` |

---

## Two packages, two jobs

```
  uv add fastapi uvicorn
```

```
  request ──►  UVICORN  ──►  FASTAPI  ──►  my function
               (server)      (framework)
               listens on    decides which
               port 8000     endpoint matches
```

- **FastAPI** = the menu and the kitchen
- **uvicorn** = the restaurant being open

⚠️ **Both are regular dependencies, NOT `--dev`.** The Dockerfile runs `uv sync --no-dev` — a web server that isn't installed can't serve anything.

---

## The code

`src/santander_cycles_forecast/api.py`:

```python
from fastapi import FastAPI

from .data import load_sample_hires
from .features import total_by_station

app = FastAPI(title="Santander Cycles Forecast")


@app.get("/health")
def health() -> dict:
    return {"status": "ok"}


@app.get("/totals")
def totals() -> list[dict]:
    df = total_by_station(load_sample_hires())
    return df.to_dict(orient="records")
```

**The decorator:**
```
  @app.get("/health")
   ▲   ▲        ▲
   │   │        the address
   │   the method (GET)
   registers the function below

  "When a GET request arrives at /health, call this."
```
**You never call these functions yourself.** FastAPI calls them.

**`/health` is a convention** — a trivial "am I alive" endpoint. Load balancers and AWS poll it to decide whether to restart your service.

**DataFrames can't travel over HTTP:**
```
       station  hires              [{"station":"Hyde Park","hires":215},
  0  Hyde Park    215     ──►       {"station":"Waterloo","hires":750}]
  1   Waterloo    750
       (pandas)              .to_dict(orient="records") → FastAPI → JSON
```

**The layering point:** `total_by_station` was written in Phase 1, tested in Phase 2, containerised in Phase 3, served in Phase 4 — **unchanged**. You put a door in front of it, you didn't rewrite it.

---

## Running locally

```
  uv run uvicorn santander_cycles_forecast.api:app --reload
                 ▲                                  ▲
                 module:object                      DEV ONLY
                 (same shape as entry points)
```

Key line: `Uvicorn running on http://127.0.0.1:8000`

**What `--reload` does on save:**
```
  WARNING:  StatReload detected changes in '...api.py'. Reloading...
  INFO:     Finished server process [44840]     ◄── old process KILLED
  INFO:     Started server process [44902]      ◄── NEW process started
```
It doesn't patch running code — it kills and restarts, and the new process reads your edited file. A supervisor process survives throughout.

**Never `--reload` in production** — a service restarting itself because a file changed is exactly wrong there.

---

## Three ways to test it

```
  BROWSER   http://127.0.0.1:8000/health
            must include :8000, or it tries port 80 and finds nothing

  /docs     http://127.0.0.1:8000/docs
            FastAPI GENERATES interactive docs from your code alone
            "Try it out" → "Execute" sends a real request

  CURL      curl http://127.0.0.1:8000/health
            works from ANY directory — it's the network, not files
```

**curl prints the raw wire format:**
```
  curl:     [{"station":"Hyde Park","hires":215},{"station":"Waterloo","hires":750}]
  browser:  the same thing, prettified by its built-in JSON viewer
```
That's a JSON **array** `[ ]` of **objects** `{ }`.

**Watch the server terminal while testing** — every request logs a line. Two terminals, one asking, one answering, is the clearest picture of a service there is.

---

## Status codes

```
  INFO:  127.0.0.1:52184 - "GET /health HTTP/1.1" 200 OK
         ▲                  ▲     ▲                ▲
         who asked          method path            status
```

| Code | Means |
|---|---|
| **200 OK** | Worked |
| **404 Not Found** | No such address. Returns `{"detail":"Not Found"}` — **JSON even for errors** |
| **422** | FastAPI's "wrong kind of data sent" |
| **500** | My code crashed |

```
  4xx = the CLIENT got it wrong      (401 unknown, 403 not allowed)
  5xx = the SERVER got it wrong
```
Same family as the GitHub 401/403 token errors.

---

## In a container

```dockerfile
CMD ["uv", "run", "uvicorn", "santander_cycles_forecast.api:app",
     "--host", "0.0.0.0", "--port", "8000"]
```

```
  docker run -p 8000:8000 santander-cycles-forecast
```

**Two flags, both fail SILENTLY if missed:**

```
   YOUR MAC                     |        CONTAINER
                                |
   curl localhost:8000 ─────────────►  uvicorn on 0.0.0.0:8000
                                |      ▲
                   -p 8000:8000 |      --host 0.0.0.0
                   opens the door      makes it listen AT the door
```

- `--host 0.0.0.0` — locally uvicorn defaults to `127.0.0.1` = "only this machine". Inside a container, "this machine" is the **container itself** → unreachable from your Mac, with no error to explain why
- `-p 8000:8000` — `hostPort:containerPort`. Containers are network-isolated; without it the port is sealed
- **No `--reload`**

**Proving the container answered** (a stray local server looks identical to curl):
```
  1. stop local uvicorn   → curl should FAIL (Connection refused)
  2. start the container  → curl succeeds
```

Or read the source address in the log:
```
  local:      INFO:  127.0.0.1:52184   - "GET /health..."
  container:  INFO:  192.168.65.1:50458 - "GET /health..."
                     ▲
                     Docker's gateway = how your Mac looks
                     FROM INSIDE the container
```

⚠️ Two processes can't share a port → `port is already allocated`. Ctrl+C the local one first.

---

## Testing endpoints in pytest

```
  uv add --dev httpx        # TestClient needs it
```

```python
from fastapi.testclient import TestClient
from santander_cycles_forecast.api import app

client = TestClient(app)


def test_health_returns_ok():
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json()["status"] == "ok"


def test_totals_returns_two_stations():
    response = client.get("/totals")
    assert response.status_code == 200
    assert len(response.json()) == 2
```

`TestClient` sends **real requests without starting a server** → runs in CI like any other test.

Same three steps, but asserting on **status codes** and **JSON bodies** instead of DataFrames.

**The dependency split working both ways:**
```
  CI:          uv sync            → INCLUDES dev → pytest + httpx present
  Dockerfile:  uv sync --no-dev   → EXCLUDES dev → neither ships
```

---

## Gotchas

| Error | Cause & fix |
|---|---|
| `Error loading ASGI app. Attribute "app" not found` | The file is **empty**. `touch src/.../api.py` made it in `src/`, then `code api.py` (bare name = *current dir*) opened a **different** file at the root — the code went there. Check VS Code's title bar; close stale tabs |
| `Invalid username or token` | **401** — expired PAT. Two-part fix: new token (tick `repo` **and** `workflow`) **and** delete the `github.com` entry in Keychain Access. Nothing shows while pasting — deliberate |
| Browser shows nothing | Missing `:8000` — it tried port 80 |
| Container API unreachable | Missing `--host 0.0.0.0` or `-p 8000:8000` |

---

## Habits

1. Second terminal for everything — the server owns the first
2. Watch the server log while testing
3. `/docs` before writing any client — it's a free test harness
4. `--reload` locally, never in the image
5. `--host 0.0.0.0` + `-p 8000:8000` whenever an API goes in a container
6. Prove *which* server answered — identical responses prove nothing about their source
