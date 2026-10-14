# Python Setup Cheat Sheet — uv & packaging

---

## The one idea: ignore the result, commit the recipe

```
  COMMIT THIS (recipe)              IGNORE THIS (result)
  ┌────────────────────┐            ┌────────────────────┐
  │ pyproject.toml     │            │ .venv/             │
  │ uv.lock            │ ──build──► │ 63 MB of packages  │
  │ .python-version    │            │ machine-specific   │
  │ ~5 KB of text      │            │ rebuildable        │
  └────────────────────┘            └────────────────────┘
```

Anyone — a colleague, a server, a Docker image — can rebuild the right side from the left side. That's "reproducible".

---

## Why isolate at all

```
  WITHOUT a virtual environment        WITH one

  ┌─────────────────┐                  ┌──────────┐  ┌──────────┐
  │  ONE global     │                  │ project A│  │ project B│
  │  Python         │                  │ pandas   │  │ pandas   │
  │  pandas 3.0.1   │                  │ 3.0.5    │  │ 2.1.0    │
  └─────────────────┘                  └──────────┘  └──────────┘
   ▲       ▲      ▲                      sealed        sealed
   │       │      │
  proj A  proj B  old notebook
  upgrade one → break the others
```

**Proved on my own machine:**
```
  uv run python -c "import pandas; print(pandas.__version__)"   → 3.0.5
  python3     -c "import pandas; print(pandas.__version__)"     → 3.0.1
```
Same laptop, same folder, two versions. Which Python you invoke decides what you get.

---

## Vocabulary

| Term | Means |
|---|---|
| **Module** | One `.py` file |
| **Package** | A folder of modules with an `__init__.py` |
| **Dependency** | A package my project needs (pandas) |
| **Transitive dependency** | A dependency *of* a dependency (pandas needs NumPy) |
| **Virtual environment** | `.venv` — a sealed folder: this project's Python + packages |
| **Editable install** | The env points at my `src/`, not a frozen copy → edits apply instantly |
| **Entry point** | A terminal command that runs one of my functions |

---

## uv commands

| Command | Does |
|---|---|
| `uv init --package <name>` | Scaffold a new project (src layout) |
| `uv sync` | Build `.venv` to exactly match `uv.lock` |
| `uv add <pkg>` | Install **+** record in `pyproject.toml` **+** update `uv.lock` |
| `uv add --dev <pkg>` | Same, but into the dev group (doesn't ship) |
| `uv remove <pkg>` | Remove and update the lock |
| `uv run <cmd>` | Run inside the env — **auto-syncs first** |
| `uv python pin 3.13` | Write `.python-version` |

**`uv add` vs `pip install`:**

```
  pip install pandas     installs it ... and tells nobody
                         repo has no record → next clone is broken

  uv add pandas          installs it
                         + writes pyproject.toml
                         + writes uv.lock
```

`uv run` is the everyday command — no "activating" environments.

---

## pyproject.toml vs uv.lock

```
  pyproject.toml                    uv.lock
  "what I asked for"                "what actually got installed"
  written by me                     written by uv
  ┌──────────────────┐              ┌────────────────────────┐
  │ pandas >= 3.0.5  │ ──resolve──► │ pandas   == 3.0.5      │
  │                  │              │ numpy    == 2.3.1      │
  │ a RANGE          │              │ tzdata   == 2025.2     │
  │ (flexible)       │              │ ...12 packages, EXACT  │
  └──────────────────┘              └────────────────────────┘
```

A range means two people installing on different days could get different versions. The lock removes that doubt.

**Commit both, together.** After `uv add`, `git status` shows both modified.

---

## pyproject.toml sections

```toml
[project]                 # name, version, requires-python, dependencies
[project.scripts]         # entry points — terminal command → function
[build-system]            # declares this builds into an installable package
[dependency-groups]       # dev-only stuff (pytest) — doesn't ship
```

**Entry points:**

```
  [project.scripts]
  santander-cycles-forecast = "santander_cycles_forecast:main"
   ▲                            ▲                         ▲
   terminal command             package                   function
```

Reads as: *"make a command with this name; when run, import that package and call `main`."*

Why it matters:
```
  a SCRIPT:   python /some/path/script.py     ← must know where the file is
  a PACKAGE:  santander-cycles-forecast       ← run by name, from anywhere
```
That second form is how a server, container or scheduler will call it.

---

## The src layout

```
  santander-cycles-forecast/
  ├── pyproject.toml
  ├── uv.lock
  ├── .python-version
  ├── README.md
  ├── tests/                       ← NOT shipped
  └── src/
      └── santander_cycles_forecast/   ← hyphens become underscores
          ├── __init__.py              (import names can't have hyphens)
          ├── data.py
          ├── features.py
          └── api.py
```

**Why the project installs itself.** Python looks for imports in two places:

```
  1. the folder I'm standing in  ──► src layout puts code OUT of reach here
  2. the installed packages area ──► so this is the ONLY way in
```

That block is deliberate:

```
  code loose in root:  works because I happened to stand there  ← a trap
  code in src/:        works only if genuinely installed        ← honest
```

The trap version breaks the moment you `cd` elsewhere, or run on a server.

**Editable install** — proved it:
```
  cd ~
  uv run --directory ~/projects/santander-cycles-forecast \
         python -c "import santander_cycles_forecast as s; print(s.__file__)"

  → /Users/fahad/projects/santander-cycles-forecast/src/.../__init__.py
     ▲                                               ▲
     worked from HOME = genuinely installed          points at MY src,
                                                     not a copy in .venv
                                                     = editable
```

---

## Module structure

One job per file:

```
  src/santander_cycles_forecast/
  ├── __init__.py   the "front door" — wiring + what's public
  ├── data.py       loading CSVs
  ├── features.py   reshaping
  ├── model.py      fitting / forecasting
  └── api.py        HTTP endpoints
```

Why, beyond tidiness: `features.py` can be tested alone in milliseconds — no data loading, no model fitting. **Untestable code is usually code that wasn't split up.**

**Importing between my own modules:**
```python
from santander_cycles_forecast.data import load_sample_hires   # absolute
from .data import load_sample_hires                            # relative — "." = my package
```
Relative inside the package (survives renames). **Absolute from `tests/`** — relative only works inside the package.

**`__init__.py` as the front door:**

```
  features.py ──┐
                ├──► __init__.py ──► what the outside world sees
  data.py     ──┘
```

```python
from .data import load_sample_hires
from .features import total_by_station

def main() -> None:
    print(total_by_station(load_sample_hires()))
```

Importing a name there attaches it to the **package**, so this works:
```python
from santander_cycles_forecast import total_by_station   # even though it lives in features.py
```
→ internal files can be renamed or split freely; nothing outside breaks.

---

## .gitignore

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

  # Editors / tools
  .vscode/
  .idea/
  .pytest_cache/
```

**Two rules:**
1. It only stops **untracked** files. Once committed, adding it later does nothing → **set it up before the first commit**
2. `git add .` is only safe **because** this exists

**Not** ignored: `uv.lock` and `.python-version` — those are recipe.

---

## Pinning Python

```
  pyproject.toml:    requires-python = ">=3.13"    ← intent (a range)
  .python-version:   3.13                          ← exact reality
```

`uv.lock` pins packages but **not the interpreter**. Without the pin:

```
  my Mac:   Python 3.13.7
  CI:       Python 3.14.7     ← newest matching ">=3.13"  ⚠️ not the same env
```

Fix: `uv python pin 3.13` → CI then ran 3.13.15. Pins the **series**, not the patch — right trade, since patches are just bug fixes.

---

## The footgun: stray markers at home

Both Git and uv search **upward** for their root marker.

```
  ~/                      ← a stray .git or pyproject.toml HERE
   ├── .git               poisons everything below
   ├── pyproject.toml
   └── projects/
       └── my-project/    ← tools latch onto the wrong root
```

**Symptoms:**
- `git status` lists your whole home directory (`../../Downloads/`)
- `uv init` says `Adding <project> as member of workspace '/Users/fahad'`

**Diagnose (read-only):**
```
  ls -ld ~/.git
  cat ~/pyproject.toml      # look for [tool.uv.workspace]
```

**Fix** — removes only the markers, never real files:
```
  rm -rf ~/.git
  rm ~/pyproject.toml ~/uv.lock ~/.python-version
```

Then from the project, `git status` should say `fatal: not a git repository` — proof the upward search finds nothing. Now `git init` **inside the project**.

**Prevention:** `pwd` before any `init`.

---

## Full setup sequence

```
  mkdir -p ~/projects && cd ~/projects
  uv init --package santander-cycles-forecast
  cd santander-cycles-forecast

  # .gitignore FIRST
  git init
  git add .
  git commit -m "Initialise project structure"

  # EMPTY repo on GitHub, then:
  git remote add origin https://github.com/<user>/<repo>.git
  git push -u origin main

  uv sync
  uv add pandas
  uv python pin 3.13
  git add pyproject.toml uv.lock .python-version
  git commit -m "Add pandas and pin Python"

  # run it, two ways:
  uv run python -c "from santander_cycles_forecast import main; main()"
  uv run santander-cycles-forecast
```

---

## Python bits

| Thing | Means |
|---|---|
| `import pandas as pd` | Load under a short alias (universal convention) |
| `def main() -> None:` | `-> None` is a **type hint** — documentation, not enforced |
| `"""Docstring."""` | Documentation attached to the function |
| `print(f)` vs `print(f(x))` | `f` shows `<function ... at 0x...>`; `f(x)` **calls** it |
| `python -c "..."` | Run a snippet straight from the shell |
| `df.groupby("station", as_index=False)["hires"].sum()` | SQL's `GROUP BY station, SUM(hires)` |
| The pandas **index** | The unlabelled `0,1,2…` column — row labels, not data |

**Two errors that look alike:**
```
  ModuleNotFoundError   → Python can't FIND the package   = environment problem
  NameError             → that name doesn't exist         = typo in my code
```

---

## Habits

1. `pwd` before any `init`
2. `.gitignore` before the first commit
3. `uv add`, never `pip install`
4. `uv run <thing>` for everything
5. One job per module; `__init__.py` is wiring only
6. Read a tool's first output line — it tells you where it thinks it is
