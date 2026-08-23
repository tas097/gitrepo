# Docker Cheat Sheet

Notes on Phase 3: turning the project into a portable image that runs identically anywhere.

---

## Why Docker (the end of the reproducibility thread)

CI proved a lot, but look at what it still needed to be true: a machine with uv, the right Python, and internet access. `uv.lock` pins my *Python packages* — it pins **nothing** about the operating system, system libraries, or surrounding tools. Plenty breaks at that layer: a C library present on one distro and not another, a missing font, a different timezone default.

**"Works in CI" is better than "works on my machine" — but it still isn't "works anywhere."**

Docker's answer: instead of shipping code and *hoping* the destination has the right environment, **ship the environment with the code.**

The thread across phases: Phase 0 isolated the project → Phase 1 made it installable → Phase 2 proved it works on a machine that isn't mine → **Phase 3 makes the machine part of the deliverable.**

---

## The core mental model

| Term | Is | Analogy |
|---|---|---|
| **Image** | A complete, frozen filesystem — OS, system libs, Python, dependencies, my code. Built once, identical every time it's used | A class / a recipe |
| **Container** | A running instance of an image | An object / a meal |
| **Dockerfile** | The plain-text recipe that builds the image. **Committed to the repo** | The written method |
| **Registry** | A public/private store of images (Docker Hub, later AWS ECR) | PyPI, but for images |

One image → many containers, all identical. The Dockerfile being a committed text file is the same idea as the CI workflow YAML: **infrastructure as code**, versioned alongside what it builds. That idea carries straight through to Terraform.

**Client vs daemon.** The `docker` command is just a **client** — a messenger. The **daemon** (a background service, provided by Docker Desktop on macOS) does the actual building and running. The client can be installed and working while the daemon isn't running, giving `Cannot connect to the Docker daemon`. `docker --version` only checks the client; `docker info` proves the client reached the daemon (look for the `Server:` section).

**On macOS, containers run inside a Linux VM.** Containers are Linux technology; Docker Desktop provides the VM. That's why `docker info` says `OSType: linux` even on a Mac.

---

## The Dockerfile, instruction by instruction

```dockerfile
FROM python:3.13-slim

COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /app

COPY pyproject.toml uv.lock .python-version README.md ./
COPY src ./src

RUN uv sync --locked --no-dev

CMD ["uv", "run", "santander-cycles-forecast"]
```

Read it as instructions for setting up a brand-new machine:

| Instruction | Does |
|---|---|
| `FROM <image>` | Start from a **base image**. `python:3.13-slim` = official Python, stripped-down variant (~hundreds of MB vs ~1.5GB for the full one). `3.13` deliberately matches `.python-version` |
| `COPY --from=<image> <src> <dest>` | Copy a file out of *someone else's* image into mine — here, borrowing the uv binary rather than installing it |
| `WORKDIR /app` | Set the working directory **inside the container**; later commands run here |
| `COPY <src> <dest>` | Copy files from my project into the image |
| `RUN <cmd>` | Execute a command **at build time** — the result is baked into the image |
| `CMD ["a", "b"]` | The **default** command run when a *container starts*. Not baked in — a default |

**`RUN` vs `CMD` is the classic confusion:**
- `RUN` → **build** time, once, result frozen into the image.
- `CMD` → **run** time, every time a container starts.

**`uv sync --locked --no-dev`:**
- `--locked` — fail if `uv.lock` is out of date. No silent re-resolving; the whole point of the lockfile.
- `--no-dev` — skip the dev dependency group, so **pytest doesn't ship**. The Phase 2 split paying off.

---

## The container has ONLY what I COPY

A container starts as a bare machine. Not my home folder, not my other files — **nothing except what the Dockerfile puts there.** `COPY` is the only door in.

**How I learned this:** the build failed with
```
failed to open file `/app/README.md`: No such file or directory
```
because `pyproject.toml` has `readme = "README.md"`, and I hadn't copied `README.md` in. It exists on my Mac; it did not exist in the container. Fix: add it to the `COPY` line.

**Proved by walking around inside:**
```
docker run -it santander-cycles-forecast bash
```
`ls` showed only the copied files — **no `tests/`, no `.git`**. `ls -a` also showed `.python-version` (copied) and `.venv` (created by `uv sync` during the build). Nothing else exists.

---

## How CMD reaches my code — the four hops

`CMD ["uv", "run", "santander-cycles-forecast"]` → my DataFrame prints. The chain:

1. **`CMD`** — Docker runs this when the container starts. Docker knows nothing beyond this.
2. **`santander-cycles-forecast` is a real executable** in the container's `.venv/bin/`, created when `uv sync` installed my package — because of `[project.scripts]` in `pyproject.toml`.
3. **That executable imports and calls** — `santander_cycles_forecast:main` reads as *"import this package, find `main`, call it."* The colon separates package from function.
4. **`main` exists** because `__init__.py` defines it. It runs `load_sample_hires()` → `total_by_station()` → prints.

Only hop 1 is Docker. Hops 2–4 are the Phase 1 packaging, working inside a container instead of on my Mac.

**`CMD` is a default, not a fixed instruction.** Anything after the image name **replaces** it:
```
docker run santander-cycles-forecast python -c "print('hello')"   # prints hello, no DataFrame
docker run -it santander-cycles-forecast bash                     # drops me into a shell inside
```
That override is how I debug a misbehaving container.

---

## Layer caching

Each instruction creates a **layer**. On rebuild, Docker reuses unchanged layers and marks them `CACHED` in the log.

**The rule:** cache reuse stops at the **first changed instruction**, and **every layer after it is invalidated too**. It never resumes.

Watched live across three builds of the same project:
| Build | Time | What happened |
|---|---|---|
| 1st | 6.2s | Nothing cached, downloaded base images |
| 2nd | 3.8s | Steps 2–3 `CACHED`; I'd edited step 4, so 4/5/6 re-ran |
| 3rd | 0.9s | **Everything** `CACHED` — nothing had changed |

**Ordering follows from the rule: put least-frequently-changed instructions first.** Dependencies change rarely; code changes constantly. If the slow dependency install sits *below* the code copy, every code edit invalidates it and reinstalls everything. Putting the install *above* the code copy lets it survive:

```dockerfile
COPY pyproject.toml uv.lock .python-version README.md ./
RUN uv sync --locked --no-dev --no-install-project   # deps only — survives code edits
COPY src ./src
RUN uv sync --locked --no-dev                        # now install my package
```
(`--no-install-project` = install the dependencies but not my package yet, since `src` isn't there at that point.) An optimisation, not a correctness issue — the simple version works fine.

**Layer sharing** also explains `docker images` output: `DISK USAGE` can far exceed `CONTENT SIZE` because images sharing a base layer store it once. Two 1.88GB images don't mean 3.76GB on disk.

---

## Build context & `.dockerignore`

```
docker build -t <name> .
```
- `-t <name>` — **tag** (name) the image. `name:tag` e.g. `santander-cycles-forecast:amd64`; defaults to `:latest`.
- `.` — the **build context**: the directory sent to the daemon.

**Two distinct stages, easily conflated:**
1. **Everything** in the context folder is *sent to the daemon* — before it even reads the `COPY` lines.
2. Only the files named in `COPY` are *put into the image*.

So `.git` and `tests/` never reach the image, but they were still being **shipped over on every build**. `.dockerignore` fixes stage 1 — same syntax as `.gitignore`:

```
.venv/
.git/
__pycache__/
.pytest_cache/
tests/
*.md
!README.md
.DS_Store
```
`!` means **except this** — exclude all markdown but keep the `README.md` the build actually needs.

Effect: `transferring context: 46.89kB` → **723B**. On a project with `.venv` unignored that's tens of MB per build. It also stops secrets or junk being accidentally baked into a published image.

`.dockerignore` is **committed** — it's recipe, not result.

---

## Architecture: arm64 vs amd64 (the AWS trap)

My Mac is Apple Silicon = **arm64** (`docker info` reports `Architecture: aarch64`). AWS Fargate typically runs **amd64** (x86_64). Images are built for a specific architecture; an arm64 image on an amd64 host fails with a confusing **`exec format error`**.

```
docker build --platform linux/amd64 -t santander-cycles-forecast:amd64 .
```

**Cached layers cannot cross architectures.** That build showed **zero `CACHED` lines** and took 7.1s vs 0.9s, re-downloading the amd64 variants of the base images (different sha256s). A layer contains compiled machine code for a specific instruction set — arm64 binaries are meaningless to an amd64 CPU. Architecture isn't a label; it goes all the way down.

The amd64 image runs **emulated** on my Mac, so it's noticeably slower locally. Expected — it's built for a machine I don't have.

---

## Everyday commands

| Command | Does |
|---|---|
| `docker --version` | Check the **client** |
| `docker info` | Check the **daemon** too — look for the `Server:` section |
| `docker build -t <name> .` | Build an image from the Dockerfile in `.` |
| `docker build --platform linux/amd64 -t <name>:amd64 .` | Build for a different architecture |
| `docker run <image>` | Create a container and start it — runs the image's `CMD` |
| `docker run <image> <cmd>` | Same, but **override** `CMD` |
| `docker run -it <image> bash` | Interactive shell **inside** the container (`-i` keep input open, `-t` give a terminal). `exit` to leave |
| `docker ps` | List **running** containers |
| `docker ps -a` | List **all** containers, including stopped ones |
| `docker images` | List local images |
| `docker container prune` | Delete **all stopped containers** (asks for confirmation) |
| `docker image prune -a` | Delete images not used by any container — heavier, be deliberate |

---

## Containers are cheap, images are heavy

**Stopped containers persist.** Stopping doesn't delete — the container keeps its filesystem changes, logs and exit status until explicitly removed. I found a `hello-world` container still sitting there from **9 months** earlier. They accumulate silently.

But pruning 17 stopped containers reclaimed only **2.675MB** — because a container is just the thin **writable layer** on top of its image, plus logs and metadata. Nearly all the bulk lives in the **images**. That's why containers can be created and destroyed freely.

**After pruning containers, `docker images` was unchanged** — deleting instances doesn't delete what they were made from (class survives, objects gone). The only visible difference: the `U` (In Use) flags disappeared.

`docker container prune` is the Docker counterpart of `git remote prune origin` — the same "sweep what's no longer needed" idea.

---

## Gotchas hit

- **`command not found: docker`** — the client isn't installed. A leftover `~/.docker` config folder is *not* evidence Docker is installed.
- **Installing isn't enough — Docker Desktop must be *running*.** That's what starts the daemon. Wait for the whale icon to stop animating. Otherwise: `Cannot connect to the Docker daemon`.
- **`failed to open file /app/<x>: No such file or directory`** — I didn't `COPY` it. The container has only what I put in it.
- **`exec format error` on a server** — architecture mismatch. Build with `--platform linux/amd64`.
- **`uv run` inside `CMD` re-syncs at container start** (visible as "Installed 6 packages" on every run). Harmless but wasteful — a production Dockerfile would call the installed executable directly rather than going through `uv run`.
- **`build_system.requires does not contain the current uv version`** — a warning, not an error; the uv version in the image is newer than the range in `pyproject.toml`.

---

## Habits worth keeping

1. `docker info` (not just `--version`) when something's wrong — it distinguishes client from daemon.
2. Read the numbered build steps: the log narrates the Dockerfile top to bottom, and shows exactly which step failed.
3. Watch for `CACHED` — it tells me what Docker reused and, by absence, what I invalidated.
4. `.dockerignore` alongside `.gitignore`, from the start.
5. `--platform linux/amd64` for anything destined for AWS.
6. `docker run -it <image> bash` to look inside — the fastest way to make a container concrete.
7. Prune stopped containers occasionally; be more deliberate about pruning images.
