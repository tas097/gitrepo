# Python Project Setup — uv & the src layout

Notes on standing up a new Python project the reproducible way, using **uv**. The throughline of the whole MLOps roadmap: **ignore the result, commit the recipe** — anyone (or any server) should be able to rebuild the environment identically from what's in the repo.

---

## The anchor idea: isolation & reproducibility

One global Python shared by every script is like committing straight to `main` with no branches — one project's package upgrade silently breaks another. The fix is to give each project its **own sealed environment**: its own Python version, its own packages, pinned to exact versions written down in files that live in the repo.

- **Ignore** the *result* — the installed packages (`.venv/`, hundreds of MB, machine-specific).
- **Commit** the *recipe* — `pyproject.toml` (what I asked for) and `uv.lock` (the exact resolved versions).

Everything later in the roadmap (Docker, CI) exists to serve this one idea.

---

## uv — one tool for three jobs

uv replaces the older `pyenv` + `pip` + `venv` stack with a single fast tool that manages **Python versions**, **virtual environments**, and **dependencies** together.

| Command | What it does |
|---|---|
| `uv --version` | Check uv is installed |
| `uv init --package <name>` | Scaffold a new project with a **src layout** (see below) |
| `uv add <pkg>` | Add a dependency — installs it *and* records it in `pyproject.toml` + `uv.lock` |
| `uv remove <pkg>` | Remove a dependency and update the lock |
| `uv sync` | Rebuild `.venv` to exactly match `uv.lock` — the reproducibility payoff |
| `uv run <cmd>` | Run a command inside the project's environment (no manual activation needed) |

(`add` / `sync` / `run` come into their own from Phase 1 onward — noted here so they're on the radar.)

---

## What `uv init --package` creates

Running `uv init --package santander-cycles-forecast` produces:

```
santander-cycles-forecast/
├── pyproject.toml        # the project's definition file
├── README.md
└── src/
    └── santander_cycles_forecast/
        └── __init__.py   # starter code with a main() stub
```

- **Hyphens → underscores:** the *project* name can have hyphens (`santander-cycles-forecast`), but the *package* folder can't — Python `import` names forbid hyphens — so uv converts them (`santander_cycles_forecast`).
- **The src layout:** code lives in `src/<package>/`, not loose in the root. Why it's the professional choice — it forces me to *install* the package before importing it, catching "it only worked because I happened to be in the right folder" bugs. Reproducibility discipline baked into the structure.
- **Git tracks files, not folders:** an empty folder is invisible to Git; it only sees the files inside — which is why the package needs that `__init__.py`.

---

## Anatomy of `pyproject.toml`

The single file that defines the project — I'll live in it through Phases 1–2.

| Section | What it's for |
|---|---|
| `[project]` | Name, version, description, `requires-python`, and the `dependencies` list |
| `[project.scripts]` | Command-line entry points — maps a command name to a `package:function` (e.g. `... = "santander_cycles_forecast:main"`) |
| `[build-system]` | Declares the project *builds* into an installable package (uv's build backend) |
| `[dependency-groups]` | Dev-only deps (e.g. test tools) kept separate from what ships |

The `[build-system]` + `[project.scripts]` lines are the fingerprints of the `--package` src layout — a plain `uv init` (no `--package`) wouldn't add them.

---

## `.gitignore` — set up BEFORE the first commit

A plain text file listing patterns Git should pretend it can't see — never show, stage, or track. It lives in the **project root**, is itself a tracked file, and travels with the repo so everyone ignores the same junk.

**Two rules that bite beginners:**
1. **`.gitignore` only stops *untracked* files.** Once a file is committed, adding it later does nothing — Git keeps tracking what it already tracks. So set it up *before* the first commit.
2. **`git add .` respects `.gitignore`.** The sweeping "add everything here" is only safe *because* the ignore file exists — it skips `.venv/`, `.DS_Store`, etc. (It also never stages the `.git` folder itself.)

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
Note what's **not** ignored: `uv.lock`. Ignore the *result* (`.venv/`); commit the *recipe* (`uv.lock`).

---

## The Phase 0 sequence (end to end)

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
```

Empty remote matters: a README on the remote creates a rival `root-commit`, giving two histories with no shared ancestor → `refusing to merge unrelated histories`. (Full explanation in the Git cheat sheet.)

---

## The footgun: stray root markers at home

**Tools search *upward* from the current folder for their root marker** — Git looks for `.git`, uv looks for `pyproject.toml`. A stray marker sitting at the home directory (`~`) poisons *every* project underneath it.

**Symptoms:**
- `git status` lists my **entire home directory** with `../../` paths → there's an accidental `.git` at `~`.
- `uv init` prints `Adding <project> as member of workspace '/Users/<me>'` → there's a stray `pyproject.toml` with a `[tool.uv.workspace]` block at `~`.

**Diagnose (read-only, changes nothing):**
```
ls -ld ~/.git            # is there an accidental repo at home?
cat ~/pyproject.toml     # is there a stray workspace file? (look for [tool.uv.workspace])
```

**Fix — removes only the stray markers, never real files** (deleting a `.git` removes *history*, not documents; the loose uv files are pure scaffolding junk):
```
rm -rf ~/.git
rm ~/pyproject.toml ~/uv.lock ~/.python-version
```
Then, from inside the project, `git status` should report `fatal: not a git repository` — proof the upward search now finds nothing stray. Re-run `git init` **inside the project** to establish the repo in the right place.

**Prevention:** always `pwd` before `git init` or `uv init`. Never run either in `~`.

---

## Habits worth keeping

1. `pwd` before any `init` command — confirm the working directory first.
2. `.gitignore` before the first commit, every time.
3. Commit the recipe (`pyproject.toml`, `uv.lock`), ignore the result (`.venv/`).
4. Read a tool's first line of output — "member of workspace…", "root-commit", "unrelated histories" are all telling me something about *where* it thinks it is.
