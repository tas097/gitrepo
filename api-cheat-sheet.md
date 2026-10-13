# API Cheat Sheet — FastAPI & serving

Notes on Phase 4: wrapping the project in an HTTP interface so other software can call it.

---

## Batch job vs service — the shift

**Batch job** — runs, produces output, **exits**. Useful, but nothing can *ask* it for anything.

**Service** — starts up and **stays running**, waiting. Something sends a request, it sends back an answer, it keeps waiting.

That's the whole of this phase. **You don't deploy a script; you deploy something that stays up.** It's what makes Phase 7's deployment meaningful.

**The visible tell:** every command before this printed output and returned my prompt. `uvicorn` starts and *never gives the prompt back* — the terminal is occupied until I stop it with Ctrl+C. That hanging terminal **is** the service. (So: second terminal window for everything else.)

---

## Vocabulary

| Term | Means |
|---|---|
| **HTTP** | The protocol web traffic uses. A **request** goes one way, a **response** comes back |
| **Endpoint** | A specific address that does one thing, e.g. `/forecast`. A service has several |
| **Method** | The verb. `GET` (fetch something), `POST` (send data in) |
| **JSON** | The standard text format for structured data in transit. Looks like a Python dict, converts cleanly |
| **Port** | A numbered door on a machine. Conventionally `8000` in development |
| **localhost / `127.0.0.1`** | My own machine talking to itself |
| **Header** | Extra metadata sent alongside a request, e.g. `accept: application/json` |

---

## FastAPI vs uvicorn — two packages, two jobs

```
uv add fastapi uvicorn
```

- **FastAPI** = the **framework**. Defines what the endpoints are and what they return.
- **uvicorn** = the **server**. Actually listens on a port and handles incoming connections.

FastAPI is the menu and the kitchen; uvicorn is the restaurant being open.

**Both are regular dependencies, NOT `--dev`.** The Dockerfile runs `uv sync --no-dev`, so anything in the dev group is **absent from the image** — and a web server that isn't installed can't serve anything. pytest stays dev (tests run in CI, never in production).

---

## Writing the API

`src/santander_cycles_forecast/api.py` — its own module, consistent with separation of concerns:

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

- **`app = FastAPI(...)`** — the application object. Everything attaches to it.
- **`@app.get("/health")`** — a **decorator** (the `@` syntax). It sits above a function and *registers* it: *"when a GET request arrives at `/health`, call the function below."* **I never call these functions myself** — FastAPI calls them when a request arrives.
- **`df.to_dict(orient="records")`** — a DataFrame can't travel over HTTP. This turns it into a list of dicts (one per row), which FastAPI serialises to JSON automatically.

**`/health` is a convention worth keeping** — a trivial endpoint confirming the service is alive. Load balancers and orchestrators (AWS in Phase 7) poll it to decide whether to restart the service.

**The layering point:** `total_by_station` was written in Phase 1, tested in Phase 2, containerised in Phase 3, and is now reachable over HTTP — **unchanged**. I didn't rewrite the logic, I put a door in front of it. The logic doesn't know whether it's called by `main()`, a test, or a web request.

---

## Running it locally

```
uv run uvicorn santander_cycles_forecast.api:app --reload
```

`module:object` — same shape as the `[project.scripts]` entry point: *"in module `santander_cycles_forecast.api`, find the object `app`."*

Key startup line: `Uvicorn running on http://127.0.0.1:8000`

**`--reload` is DEVELOPMENT ONLY.** It watches files and restarts on every save:
```
WARNING:  StatReload detected changes in '...api.py'. Reloading...
INFO:     Finished server process [44840]
INFO:     Started server process [44902]
```
Note the **process IDs change** — it doesn't patch running code, it **kills the process and starts a fresh one**, which then reads the edited file. A separate supervisor process (the reloader) survives throughout.

Never use `--reload` in production — a service restarting itself because a file changed on disk is exactly what's not wanted there.

---

## Testing endpoints

Three ways, all hitting the same running server:

**Browser** — `http://127.0.0.1:8000/health`. Must include the **port**; without `:8000` the browser tries port 80 and finds nothing.

**`/docs`** — FastAPI **generates interactive documentation from the code alone**. Lists every endpoint, infers response shapes from type hints, and "Try it out" → "Execute" sends a real request. No documentation written by hand.

**curl** — a command-line HTTP client:
```
curl http://127.0.0.1:8000/health
curl -X 'GET' 'http://127.0.0.1:8000/health' -H 'accept: application/json'
```
- `-X 'GET'` — the method (GET is the default, usually omitted)
- `-H '...'` — a header
- trailing `\` — line continuation, nothing more

curl works **from any directory** — it's talking over the network, not to files. And it prints the **raw wire format**, one dense line:
```
[{"station":"Hyde Park","hires":215},{"station":"Waterloo","hires":750}]
```
The browser's prettier display is its built-in JSON viewer. This is what actually travelled: a JSON **array** `[]` of **objects** `{}`.

**Watch the server terminal while testing** — every request logs a line. Making requests in one terminal and watching them arrive in another is the clearest possible picture of a service.

---

## Status codes

The server log ends each line with one: `"GET /health HTTP/1.1" 200 OK`

| Code | Means |
|---|---|
| **200 OK** | Worked |
| **404 Not Found** | No such address. Even the error comes back as **JSON** (`{"detail":"Not Found"}`) — an API behaving properly |
| **422** | FastAPI's "you sent the wrong kind of data" (shows up once endpoints take input) |
| **500 Internal Server Error** | My code crashed handling the request |

**4xx = the client got something wrong. 5xx = the server did.** Same system as the GitHub **401** (don't know who you are) and **403** (know you, not allowed) from the token troubles.

---

## Containerising the API

```dockerfile
CMD ["uv", "run", "uvicorn", "santander_cycles_forecast.api:app", "--host", "0.0.0.0", "--port", "8000"]
```

```
docker build -t santander-cycles-forecast .
docker run -p 8000:8000 santander-cycles-forecast
```

**Two flags that are easy to miss, and both fail silently:**

**`--host 0.0.0.0`** — locally uvicorn defaults to `127.0.0.1`, meaning "only accept connections from this machine." Inside a container, "this machine" is **the container itself**, so the service is unreachable from my Mac even with the port published. `0.0.0.0` = accept on any network interface. Without it the API looks completely dead from outside, with nothing in the logs explaining why.

**`-p 8000:8000`** — **publish a port**, `hostPort:containerPort`. Containers are network-isolated by default; without this the port is sealed. Left number = the door on my Mac, right = the door inside the container.

**No `--reload`** in the container — development only.

**How to prove the container is answering** (and not a stray local server — both look identical to curl):
1. Stop the local uvicorn → `curl` should **fail** with `Connection refused`
2. Start the container → `curl` succeeds

Or read the source address in the container log:
```
INFO:  192.168.65.1:50458 - "GET /health HTTP/1.1" 200 OK
```
`192.168.65.1` is Docker's internal network gateway — how my Mac appears **from inside the container**. Locally it logs `127.0.0.1`. Different address = the request genuinely crossed the boundary.

Also: `Uvicorn running on http://0.0.0.0:8000` (vs `127.0.0.1` locally) confirms `--host` took effect.

**Two processes can't share a port** — if the local server is still running, `docker run -p 8000:8000` fails with `port is already allocated`. Ctrl+C the local one first.

---

## Testing the API in pytest

```
uv add --dev httpx      # TestClient needs it
```

`tests/test_api.py`:
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

**`TestClient` sends real requests without starting a server**, so these run in CI like any other test. Same three-step shape (set up → run → assert), but asserting on **status codes** and **JSON bodies** rather than DataFrames.

Note the dependency split working in both directions: CI runs `uv sync` (**includes** dev, so httpx and pytest are there), the Dockerfile runs `uv sync --no-dev` (**excludes** them). Same lockfile, two different installs, by design.

---

## Gotchas hit

- **`Error loading ASGI app. Attribute "app" not found in module ...`** — the module exists but is **empty**. Cause: `touch src/.../api.py` created the file in `src/`, then `code api.py` (bare filename = *current directory*) opened a **different** file at the project root, and the code got pasted there. Same trap as the pytest `test_features.py` collision. **Check VS Code's title bar shows the path I expect**, and close stale tabs after moving files.
- **`Invalid username or token. Password authentication is not supported`** — the **401 family**: GitHub doesn't recognise the credential (vs **403**, which is "recognised but not permitted"). After a long gap this almost always means the **PAT expired**. Fix is two-part: generate a new token (ticking **both** `repo` and `workflow`), **and** delete the stale `github.com` entry from Keychain Access — otherwise Git keeps sending the dead one. Nothing appears on screen while pasting the token; that's deliberate.
  - Longer-term fix: **Git Credential Manager** (`brew install --cask git-credential-manager`) — browser login, handles renewal.
- **Hitting the port without `:8000`** — the browser tries port 80 and finds nothing.

---

## Habits worth keeping

1. Second terminal for anything else — the server owns the first one.
2. Watch the server log while testing; requests arriving in real time is the best feedback loop there is.
3. `/docs` before writing a client — it's a free test harness.
4. `--reload` locally, never in the image.
5. `--host 0.0.0.0` + `-p 8000:8000` whenever an API goes in a container.
6. Prove *which* server answered — stop one, confirm the failure, start the other. Identical responses prove nothing about their source.
