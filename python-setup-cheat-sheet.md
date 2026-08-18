# Python Project Setup — uv, packaging & the src layout

Notes on standing up a Python project the reproducible way, using **uv**. The throughline of the whole MLOps roadmap: **ignore the result, commit the recipe** — anyone (or any server) should be able to rebuild the environment identically from what's in the repo.

---

## The anchor idea: isolation & reproducibility

One global Python shared by every script is like committing straight to `main` with no branches — one project's package upgrade silently breaks another. The fix is to give each project its **own sealed environment**: its own Python version, its own packages, pinned to exact versions written down in files that live in the repo.

- **Ignore** the *result* — the installed packages (`.venv/`, machine-specific, tens/hundreds of MB).
- **Commit** the *recipe* — `pyproject.toml` (what I asked for) and `uv.lock` (the exact resolved versions).

Everything later in the roadmap (Docker, CI) exists to serve this one idea.

---

## Vocabulary (these get used loosely — worth being precise)

| Term | Means |
|---|---|
| **Module** | A single `.py` file. `data.py` is a module. |
| **Package** | A *folder* of modules with an `__init__.py` in it — that file is what marks the folder importable. |
| **Import** | Python loading code from another module: `import pandas`, `from .data import load_sample_hires`. |
| **Dependency** | A package my project needs (e.g. pandas). |
| **Transitive dependency** | A dependency *of* a dependency (pandas needs NumPy). Installed, but not listed in my `pyproject.toml`. |
| **Virtual environment** | A self-contained folder (`.venv`) holding a Python interpreter + exactly this project's packages. |
| **Editable install** | The environment points back at my `src/` folder rather than holding a frozen copy — edits take effect immediately. |
| **Entry point** | A terminal command created by config that runs one of my functions. |

---

## uv — one tool for three jobs

uv replaces the older `pyenv` + `pip` + `venv` stack with a single fast tool that manages **Python versions**, **virtual environments**, and **dependencies** together.

| Command | What it does |
|---|---|
| `uv --version` | Check uv is installed |
| `uv init --package <name>` | Scaffold a new project with a **src layout** |
| `uv sync` | Build/refresh `.venv` so it exactly matches `uv.lock` — what a *new machine* runs after cloning |
| `uv add <pkg>` | Install a dependency **and** record it in `pyproject.toml` + `uv.lock` |
| `uv remove <pkg>` | Remove a dependency and update the lock |
| `uv run <cmd>` | Run a command inside the project's environment — **auto-syncs first**, so it builds/fixes `.venv` if needed |

**Why `uv add` beats `pip install`:** `pip install pandas` installs it and *nothing else* — nothing in the repo records that the project needs pandas, so the next person to clone gets a broken project. `uv add pandas` does three things at once: installs it, adds it to `pyproject.toml`, updates `uv.lock`. The recipe stays honest automatically. That silent drift is the classic Python reproducibility failure.

**`uv run` is the everyday command.** No manual environment "activation" needed — it runs things inside the project environment and syncs first.

---

## `pyproject.toml` vs `uv.lock` — intent vs exact reality

Both are committed. They do different jobs:

| File | Role | Written by | Contains |
|---|---|---|---|
| `pyproject.toml` | **What I asked for** — intent | Me (and `uv add`) | Direct dependencies only, usually as *ranges* (`pandas>=3.0.5`) |
| `uv.lock` | **What was actually resolved** | uv, automatically | *Every* package incl. transitive ones, each pinned to one **exact** version |

**Why both?** A range like `>=3.0.5` is deliberately flexible so I can upgrade later — but flexible means two people installing on different days could get *different* versions. The lockfile removes that ambiguity: `uv sync` six months from now rebuilds the identical environment rather than whatever's newest. **Intent in one file, exact reality in the other. That pair is what "reproducible" means in practice.**

Commit them **together** — after `uv add`, `git status` shows `pyproject.toml` modified and `uv.lock` modified/untracked. Recipe and resolution travel as a pair.

---

## Anatomy of `pyproject.toml`

| Section | What it's for |
|---|---|
| `[project]` | Name, version, description, `requires-python`, and the `dependencies` list |
| `[project.scripts]` | **Entry points** — maps a terminal command to a `package:function` |
| `[build-system]` | Declares the project *builds* into an installable package (uv's build backend) |
| `[dependency-groups]` | Dev-only deps (e.g. test tools) kept separate from what ships |

The `[build-system]` + `[project.scripts]` lines are the fingerprints of the `--package` src layout — a plain `uv init` wouldn't add them.

**Entry points, concretely:**
```toml
[project.scripts]
santander-cycles-forecast = "santander_cycles_forecast:main"
```
Read as: *"create a terminal command `santander-cycles-forecast`; when run, import the package `santander_cycles_forecast` and call its `main` function."* The colon separates package from function. On install, uv generates a small executable in `.venv/bin/`.

**Why this matters:** a *script* is run by knowing where the file is (`python some/path/script.py`). A *packaged* project is run **by name, from anywhere**, with no idea where the file lives — which is exactly how a server, container, or scheduler will invoke it later.

---

## What `uv init --package` creates

```
santander-cycles-forecast/
├── pyproject.toml        # the project's definition file
├── README.md
└── src/
    └── santander_cycles_forecast/
        └── __init__.py   # starter code with a main() stub
```

- **Hyphens → underscores:** the *project* name can have hyphens (`santander-cycles-forecast`), but the *package* folder can't — Python `import` names forbid them — so uv converts (`santander_cycles_forecast`).
- **Git tracks files, not folders:** an empty folder is invisible to Git; it only sees the files inside — which is why the package needs that `__init__.py`.

---

## Why a project installs itself (the src layout payoff)

When I write `import santander_cycles_forecast`, Python searches a specific list of places: roughly **the folder I'm standing in**, then **the installed-packages area of the environment**. Not found there → `ModuleNotFoundError`.

With a src layout, my code sits in `src/<package>/` — *not* in the folder I stand in. So "look around the current directory" **can't** find it. The only route in is for the project to be **installed** into the environment.

**That block is deliberate.** Without `src/` (code loose in the root), imports work "for free" whenever I happen to be standing in that folder — which looks convenient and is a trap: the code appears to work, but only because of *where I was standing*. Move directory, or run it on a server or in Docker, and it breaks. The src layout makes that luck impossible: **if it imports, it's genuinely installed, and it'll behave the same anywhere.**

uv installs it **editable** — a pointer back to `src/`, not a frozen copy. So proper installation *and* instant edits: new modules and code changes are picked up with no reinstall, no re-sync.

**Proving both at once** — run from a totally unrelated directory:
```
cd ~
uv run --directory ~/projects/<project> python -c "import <package>; print(<package>.__file__)"
```
- It **succeeding** from `~` proves the package is genuinely installed (location-independent).
- `__file__` printing my real `src/...` path (not something inside `.venv/lib/.../site-packages/`) proves the install is **editable**.

---

## Virtual environments in practice

`.venv` is the project's sealed environment. It's in `.gitignore` — the *result*, not the *recipe*.

**Seeing isolation for real:**
```
uv run python -c "import pandas; print(pandas.__version__)"   # project env  → 3.0.5
python3 -c "import pandas; print(pandas.__version__)"         # global Python → 3.0.1
```
Same machine, same folder, **two different versions**, depending only on which Python was invoked. The project is pinned by `uv.lock` and unaffected by whatever's installed globally; equally, nothing done in the project can disturb the global one older notebooks rely on.

Without this, one pandas serves everything: upgrading for this project silently breaks an old analysis, and code "works on my Mac" because of a version installed years ago — then fails on a server. That whole class of bug is what this eliminates.

**Size check:** `du -sh .venv` (`-s` summarise to one total, `-h` human-readable). Empty env ≈ 116K (uv **symlinks** a shared interpreter rather than copying it). After `uv add pandas` ≈ 63M — a ~550× jump from one library. That number *is* the argument for `.gitignore`.

---

## `.gitignore` — set up BEFORE the first commit

A plain text file listing patterns Git should pretend it can't see. Lives in the **project root**, is itself a tracked file, and travels with the repo so everyone ignores the same junk.

**Two rules that bite:**
1. **`.gitignore` only stops *untracked* files.** Once a file is committed, adding it later does nothing. Set it up *before* the first commit.
2. **`git add .` respects `.gitignore`.** The sweeping "add everything" is only safe *because* the ignore file exists. (It also never stages `.git` itself.)

**A Python + macOS starter:**
```
# Python
__pycache__/
*.py[cod]
*.egg-info/
build/
dist/

# Virtual environment
.venv/

# Jupyter
.ipynb_checkpoints/

# Secrets — NEVER commit
.env

# macOS
.DS_Store

# Editors
.vscode/
.idea/
```
Note what's **not** ignored: `uv.lock`. Ignore the result (`.venv/`), commit the recipe.

---

## Module structure — separation of concerns

One file is fine for a stub and wrong for a real project: everything can touch everything, and testing any piece means running the lot. Split so each module owns **one job**:

```
src/santander_cycles_forecast/
├── __init__.py    # the "front door" — wiring + what the package exposes
├── data.py        # loading raw TfL CSVs
├── features.py    # reshaping into model-ready form
└── model.py       # fitting and forecasting
```

**Why it matters beyond tidiness:** `features.py` can be tested on its own in milliseconds — no data loading, no model fitting. **Untestable code is almost always code that hasn't been split up.** This is the prerequisite for Phase 2.

**Importing between my own modules** — two forms:
```python
from santander_cycles_forecast.data import load_sample_hires   # absolute — full path from package root
from .data import load_sample_hires                            # relative — "." = the package I'm in
```
Both work. **Relative is conventional inside a package** — tidier, and survives a package rename. Either only works because the package is properly installed.

**`__init__.py` as the front door.** Importing a name there attaches it to the **package**, not just that file:
```python
from .data import load_sample_hires
from .features import total_by_station

def main() -> None:
    print(total_by_station(load_sample_hires()))
```
So `from santander_cycles_forecast import total_by_station` works even though it's *defined* in `features.py`. **`__init__.py` decides the public surface** — internal files can be renamed, split, or reorganised freely and nothing that depends on the package breaks. Keep it free of logic: imports plus a thin `main()`.

---

## Python bits worth remembering

| Thing | Means |
|---|---|
| `import pandas as pd` | Load a library under a short alias (`pd` is universal convention) |
| `def main() -> None:` | Define a function; `-> None` is a **type hint** — documentation for humans/tools, not enforced at runtime |
| `"""Docstring."""` | Documentation attached to the function itself; tools can read it |
| `print(f)` vs `print(f(x))` | `print(total_by_station)` shows `<function ... at 0x...>` — the function *object*. Add `(...)` to actually **call** it. Functions are values that can be passed around |
| `python -c "..."` | Run a snippet of Python directly from the shell — handy for quick checks |
| `df.groupby("station", as_index=False)["hires"].sum()` | pandas for SQL's `GROUP BY station, SUM(hires)`. `as_index=False` keeps `station` a normal column instead of moving it to the index |
| The pandas **index** | The unlabelled leftmost `0,1,2…` column — row labels pandas maintains, *not* data. Later set to the date for time-series work |

**Two error types that look alike but aren't:**
- **`ModuleNotFoundError`** — Python couldn't find a package to import. An *environment* problem (not installed / wrong environment).
- **`NameError: name 'x' is not defined`** — referred to a name that doesn't exist. A *typo or logic* problem in my code.

---

## The Phase 0 → 1 sequence (end to end)

```
# 1. Make a home for projects and move into it (NOT the home dir — see footgun below)
mkdir -p ~/projects && cd ~/projects

# 2. Scaffold with a src layout
uv init --package santander-cycles-forecast
cd santander-cycles-forecast

# 3. Version control — .gitignore FIRST, then the first commit
git init
# create/replace .gitignore (Python + macOS) before staging
git add .
git commit -m "Initialise project structure with uv (src layout) and Python .gitignore"

# 4. Push to a NEW, EMPTY GitHub repo (no README / .gitignore / licence ticked)
git remote add origin https://github.com/<user>/santander-cycles-forecast.git
git push -u origin main

# 5. Build the environment and add the first dependency
uv sync
uv add pandas
git add pyproject.toml uv.lock && git commit -m "Add pandas dependency"

# 6. Run it — two equivalent ways
uv run python -c "from santander_cycles_forecast import main; main()"
uv run santander-cycles-forecast          # via the [project.scripts] entry point
```

Empty remote matters: a README on the remote creates a rival `root-commit`, giving two histories with no shared ancestor → `refusing to merge unrelated histories`. (Full explanation in the Git cheat sheet.)

---

## The footgun: stray root markers at home

**Tools search *upward* from the current folder for their root marker** — Git looks for `.git`, uv looks for `pyproject.toml`. A stray marker at the home directory (`~`) poisons *every* project underneath it.

**Symptoms:**
- `git status` lists my **entire home directory** with `../../` paths → an accidental `.git` at `~`.
- `uv init` prints `Adding <project> as member of workspace '/Users/<me>'` → a stray `pyproject.toml` with a `[tool.uv.workspace]` block at `~`.

**Diagnose (read-only):**
```
ls -ld ~/.git            # accidental repo at home?
cat ~/pyproject.toml     # stray workspace file? (look for [tool.uv.workspace])
```

**Fix — removes only the stray markers, never real files** (deleting a `.git` removes *history*, not documents; the loose uv files are pure scaffolding):
```
rm -rf ~/.git
rm ~/pyproject.toml ~/uv.lock ~/.python-version
```
Then from inside the project, `git status` should report `fatal: not a git repository` — proof the upward search now finds nothing stray. Re-run `git init` **inside the project**.

**Prevention:** always `pwd` before `git init` or `uv init`. Never run either in `~`.

---

## Habits worth keeping

1. `pwd` before any `init` command — confirm the working directory first.
2. `.gitignore` before the first commit, every time.
3. Commit the recipe (`pyproject.toml`, `uv.lock`), ignore the result (`.venv/`).
4. `uv add`, never `pip install` — keeps the recipe honest.
5. `uv run <thing>` for everything — it syncs first, so the environment is never stale.
6. One job per module; keep `__init__.py` as wiring only.
7. Read a tool's first line of output — "member of workspace…", "root-commit", "unrelated histories" all say something about *where* it thinks it is.