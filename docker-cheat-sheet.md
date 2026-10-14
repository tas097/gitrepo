# Docker Cheat Sheet

---

## Why — the last gap

```
  uv.lock pins:        pandas, numpy, ...        ✅
  uv.lock does NOT:    the OS                    ❌
                       system libraries          ❌
                       installed tools           ❌
```

Plenty breaks at that layer: a C library missing on another distro, a different timezone default.

```
  "works on my machine"  →  "works in CI"  →  "works anywhere"
                                                    ▲
                                                 Docker
```

**Instead of shipping code and hoping the destination has the right environment, ship the environment with the code.**

The thread: Phase 0 isolated it → Phase 1 made it installable → Phase 2 proved it works elsewhere → **Phase 3 makes the machine part of the deliverable.**

---

## Image vs container

```
  Dockerfile  ──build──►   IMAGE   ──run──►   CONTAINER
  (the recipe)             (frozen)           (running)
                              │
                              ├──run──► container 1
                              ├──run──► container 2
                              └──run──► container 3
                                        all identical
```

| Term | Is | Like |
|---|---|---|
| **Dockerfile** | The text recipe. **Committed** | The written method |
| **Image** | A frozen filesystem: OS + Python + deps + my code | A class / a recipe |
| **Container** | A running instance of an image | An object / a meal |
| **Registry** | A store of images (Docker Hub, later AWS ECR) | PyPI, for images |

---

## Client vs daemon

```
  docker CLI  ───asks───►  DAEMON  ───does the work
  (messenger)              (Docker Desktop on macOS)
```

```
  docker --version   → checks the CLIENT only
  docker info        → proves it reached the DAEMON (look for "Server:")
```

Installing isn't enough — **Docker Desktop must be running**. Otherwise: `Cannot connect to the Docker daemon`.

On macOS, containers run inside a **Linux VM** that Docker Desktop provides. That's why `docker info` says `OSType: linux` on a Mac.

---

## The Dockerfile

```dockerfile
FROM python:3.13-slim                                      # 1

COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv   # 2

WORKDIR /app                                               # 3

COPY pyproject.toml uv.lock .python-version README.md ./   # 4
COPY src ./src                                             # 5

RUN uv sync --locked --no-dev                              # 6

CMD ["uv", "run", "santander-cycles-forecast"]             # 7
```

Read it as setting up a brand-new machine:

```
  1. FROM     start from a base machine (Python already on it)
  2. COPY     borrow the uv tool from someone else's image
  3. WORKDIR  work in /app from here on
  4. COPY     put my recipe files on it
  5. COPY     put my code on it
  6. RUN      install dependencies   ← BUILD time, baked in
  7. CMD      what to run on start   ← RUN time, every container
```

**`RUN` vs `CMD` — the classic confusion:**
```
  RUN  ──► happens ONCE, at build, frozen into the image
  CMD  ──► happens EVERY time a container starts
```

**`uv sync --locked --no-dev`:**
```
  --locked   fail if uv.lock is out of date (no silent re-resolving)
  --no-dev   skip the dev group → pytest does NOT ship
```

`python:3.13-slim` — slim is stripped-down (~hundreds of MB vs ~1.5 GB full). `3.13` matches `.python-version`.

---

## The container has ONLY what you COPY

```
  MY MAC                          CONTAINER
  ├── .git/           ✗           /app/
  ├── tests/          ✗           ├── pyproject.toml   ✓ copied
  ├── .venv/          ✗           ├── uv.lock          ✓ copied
  ├── pyproject.toml  ✓ ──────►   ├── .python-version  ✓ copied
  ├── uv.lock         ✓ ──────►   ├── README.md        ✓ copied
  ├── README.md       ✓ ──────►   ├── src/             ✓ copied
  └── src/            ✓ ──────►   └── .venv/           ← made by uv sync
```

**How I learned it:**
```
  failed to open file `/app/README.md`: No such file or directory
```
`pyproject.toml` had `readme = "README.md"`, and I hadn't copied it. It exists on my Mac; it did not exist in the container.

**Proved by looking inside:**
```
  docker run -it santander-cycles-forecast bash
  root@ebbf8e83c526:/app#  ls
  README.md  pyproject.toml  src  uv.lock      ← no tests/, no .git
```

---

## How CMD reaches my code

```
  CMD ["uv","run","santander-cycles-forecast"]
       │
       ▼
  .venv/bin/santander-cycles-forecast        ← made by uv sync, because of
       │                                        [project.scripts]
       ▼
  import santander_cycles_forecast ; call main()
       │
       ▼
  main() → load_sample_hires() → total_by_station() → print
```

Only the first hop is Docker. The rest is Phase 1 packaging, running in a container instead of on my Mac.

**`CMD` is a default, not fixed.** Anything after the image name **replaces** it:
```
  docker run <image>                      → runs CMD
  docker run <image> python -c "print(1)" → runs that instead
  docker run -it <image> bash             → drops me into a shell inside
```

---

## Layer caching

Each instruction = one **layer**. On rebuild, unchanged layers are reused and marked `CACHED`.

**The rule:**
```
  step 1  ✓ CACHED
  step 2  ✓ CACHED
  step 3  ✗ CHANGED  ◄── first change
  step 4  ✗ re-runs   ┐
  step 5  ✗ re-runs   ├── everything below re-runs too
  step 6  ✗ re-runs   ┘
```

Cache stops at the first change and **never resumes**.

**Watched live:**
```
  build 1   6.2s   nothing cached, downloading base images
  build 2   3.8s   edited step 4 → steps 2-3 cached, 4/5/6 re-ran
  build 3   0.9s   nothing changed → ALL cached
  build 4   7.1s   --platform linux/amd64 → ZERO cached
```

**Ordering = the rule applied with foresight.** Put least-changing first:

```
  SLOW VERSION                      FAST VERSION
  COPY config                       COPY config
  COPY src          ◄── edits here  RUN uv sync (deps only)  ◄── survives
  RUN uv sync       ◄── re-runs!    COPY src                 ◄── edits here
                                    RUN uv sync (my package) ◄── fast
```

Dependencies change rarely; code changes constantly.

**Layer sharing** also explains `docker images`: `DISK USAGE` can far exceed `CONTENT SIZE`, because images sharing a base layer store it once.

---

## Build context & .dockerignore

```
  docker build -t santander-cycles-forecast .
                ▲                           ▲
                tag (name)                  build context
```

**Two stages people merge together:**

```
  stage 1: EVERYTHING in the folder is sent to the daemon
           (before it even reads your COPY lines)
                         │
                         ▼
  stage 2: only files named in COPY go INTO the image
```

So `.git` never reached the image — but it was still **shipped over every build**. `.dockerignore` fixes stage 1:

```
  .venv/
  .git/
  __pycache__/
  .pytest_cache/
  tests/
  *.md
  !README.md          ← "!" means EXCEPT this
  .DS_Store
```

```
  transferring context: 46.89kB   ──►   723B
```

`.dockerignore` is **committed** — recipe, not result.

---

## Architecture: the AWS trap

```
  MY MAC (Apple Silicon)        AWS Fargate
  arm64 / aarch64               amd64 / x86_64
         │                            ▲
         └──── image built here ──────┘
                                 ✗ exec format error
```

Fix:
```
  docker build --platform linux/amd64 -t santander-cycles-forecast:amd64 .
                                                                  ▲
                                                                  tag it separately
```

**Cached layers can't cross architectures** — that build showed **zero `CACHED`** lines and re-downloaded the base images. A layer holds compiled machine code for one instruction set; arm64 binaries are meaningless to an amd64 CPU.

The amd64 image runs **emulated** on a Mac → noticeably slower. Expected.

---

## Commands

| Command | Does |
|---|---|
| `docker --version` | Check the client |
| `docker info` | Check the daemon too |
| `docker build -t <name> .` | Build an image |
| `docker build --platform linux/amd64 -t <name>:amd64 .` | Build for another architecture |
| `docker run <image>` | Create + start a container (runs `CMD`) |
| `docker run <image> <cmd>` | Same, but **override** `CMD` |
| `docker run -it <image> bash` | Shell **inside** the container (`exit` to leave) |
| `docker run -p 8000:8000 <image>` | Publish a port (`host:container`) |
| `docker ps` | **Running** containers |
| `docker ps -a` | **All** containers, incl. stopped |
| `docker images` | Local images |
| `docker container prune` | Delete all stopped containers |
| `docker image prune -a` | Delete unused images — heavier, be deliberate |

---

## Containers are cheap, images are heavy

```
  IMAGE  (1.88 GB)            CONTAINER
  ┌──────────────────┐        ┌──────────────────┐
  │ read-only layers │        │ thin WRITABLE    │  ← only this is the container
  │ OS + Python +    │◄───────│ layer + logs     │     (~a few hundred KB)
  │ deps + my code   │        └──────────────────┘
  └──────────────────┘
```

- **Stopped containers persist** — I found a `hello-world` still there from **9 months** earlier
- Pruning 17 of them reclaimed only **2.675 MB**
- After pruning, `docker images` was **unchanged** — deleting instances doesn't delete the image (class survives, objects gone). Only the `U` (In Use) flags disappeared

`docker container prune` is the Docker version of `git remote prune origin` — sweep what's no longer needed.

---

## Gotchas

| Error | Cause & fix |
|---|---|
| `command not found: docker` | Not installed. A leftover `~/.docker` folder is **not** evidence it is |
| `Cannot connect to the Docker daemon` | Docker Desktop isn't **running**. Wait for the whale to settle |
| `failed to open file /app/<x>` | Didn't `COPY` it. The container has only what you put in |
| `exec format error` | Architecture mismatch → `--platform linux/amd64` |
| `port is already allocated` | Something else is on that port. Ctrl+C it first |
| `uv run` re-syncs on every container start | Wasteful, not wrong. Production would call the installed executable directly |
| `build_system.requires does not contain the current uv version` | A warning, not an error |

---

## Habits

1. `docker info`, not just `--version`, when something's wrong
2. Read the numbered build steps — the log narrates the Dockerfile and shows which step failed
3. Watch for `CACHED` — it shows what you reused and, by absence, what you invalidated
4. `.dockerignore` alongside `.gitignore`, from the start
5. `--platform linux/amd64` for anything going to AWS
6. `docker run -it <image> bash` to look inside — fastest way to make it concrete
