# Testing & CI Cheat Sheet — pytest and GitHub Actions

Notes on Phase 2: writing tests, and getting them to run automatically on every pull request.

---

## Why bother

Without tests, the only way to know code works is to run it and eyeball the output. That doesn't scale: it's slow, I'll forget to check the thing I broke, and in six months I won't remember what "correct" looked like. A test **encodes the answer once, permanently**, and re-checks it in milliseconds forever.

The quieter benefit: tests are the safety net that makes me *willing* to change code. Without them, refactoring is nerve-wracking. With them, I change something, run the tests, and know immediately.

**A test I've never seen fail is unproven.** Plenty of tests pass because they're accidentally asserting nothing. Break the assertion deliberately, watch it fail, then fix it — the single most valuable habit in this phase. Same applies to CI: a gate I've never seen block anything isn't proven.

---

## The core idea

A test is a function that does three things:

1. **Set up** a known input.
2. **Run** my code on it.
3. **Assert** the result equals what I expect.

`assert` is a Python keyword meaning *"this must be true — if it isn't, fail loudly."* That's genuinely the whole concept; the rest is convention and tooling.

Keep the input **deliberately tiny** — a test should be obviously correct by inspection. If I have to think hard about what the expected answer is, the test is too big.

---

## pytest — setup and conventions

**Install as a dev dependency** — pytest is needed to *develop* the project, not to *run* it. Installing a test framework into production is dead weight.

```
uv add --dev pytest
```

The `--dev` flag puts it in `[dependency-groups]` rather than `[project] dependencies`:

```toml
dependencies = [
    "pandas>=3.0.5",      # ships with the project
]

[dependency-groups]
dev = [
    "pytest>=9.1.1",      # only needed to develop it
]
```
`uv.lock` updates too — it records *everything* installed, dev tools included.

**Where tests live:** a `tests/` folder at the **project root**, *outside* `src/`. Tests aren't part of the package I ship — nobody installing the tool wants my test suite. Same shipped-vs-development split as the dependency groups.

```
project/
├── src/<package>/     # the code that ships
└── tests/             # NOT shipped
    └── test_features.py
```

**Discovery is by naming convention** — pytest looks for:
- files named `test_*.py`
- functions inside them named `test_*`

Miss the prefix and pytest **silently ignores** the test. Classic first-day confusion.

**Tests import the installed package, absolutely:**
```python
from santander_cycles_forecast.features import total_by_station
```
Absolute, not relative — relative imports (`.features`) only work *inside* the package, and `tests/` is outside it. This working from a directory outside `src/` is the src layout paying off.

---

## Writing a test

```python
import pandas as pd

from santander_cycles_forecast.features import total_by_station


def test_total_by_station_sums_hires_per_station():
    df = pd.DataFrame(                       # 1. set up a known input
        {
            "station": ["Hyde Park", "Waterloo", "Hyde Park"],
            "hires": [100, 50, 20],
        }
    )

    result = total_by_station(df)            # 2. run the real code

    assert result.loc[result["station"] == "Hyde Park", "hires"].iloc[0] == 120
    assert len(result) == 2                  # 3. assert the expected answer
```

**Name tests descriptively** — `test_total_by_station_sums_hires_per_station` reads as a sentence stating what should be true. The name is documentation; it's what shows up when it fails.

**Unpacking that pandas assertion:**
- `result["station"] == "Hyde Park"` → a column of True/False
- `result.loc[condition, "hires"]` → the `hires` value(s) where the condition holds (pandas' `WHERE`)
- `.iloc[0]` → the first of those, by position

---

## Running pytest & reading its output

```
uv run pytest          # discovers tests itself — no file argument needed
```

**Passing:**
```
collected 1 item

tests/test_features.py .                    [100%]

1 passed in 0.28s
```
- `collected N items` — how many test functions discovery found.
- The characters after the filename are the results — **one per test**: `.` pass, `F` fail, `E` error. Twelve passing tests = twelve dots.
- `[100%]` is progress through the suite.

**Failing — pytest's best feature:**
```
tests/test_features.py F                    [100%]

    def test_total_by_station_sums_hires_per_station():
        ...
>       assert result.loc[...].iloc[0] == 120
E       assert np.int64(120) == 999

tests/test_features.py:16: AssertionError
```
It doesn't just say "failed". It:
- reprints the whole test function (see the setup without opening the file)
- marks the failing line with `>`
- **evaluates the expression and shows the actual value** — `E assert np.int64(120) == 999`
- gives the exact file:line

That actual-value line is the gift: in real debugging I don't know the actual value — that's the problem. Half the work is done before I open anything.

**Type tells:** `np.int64` / `np.float64` just means pandas stores numbers in NumPy types; they compare to plain numbers normally. But a type *changing* is a hint — e.g. `.sum()` → `int64` but `.mean()` → `float64`, so a stray float can point at what went wrong.

---

## pytest gotchas hit

- **`import file mismatch` / two files with the same basename.** pytest imports test files as modules, so `test_features.py` in *two* places (e.g. project root and `tests/`) collide under the same module name. The error prints **both conflicting paths** — read them. Fix: delete or `mv` the stray one. Give tests distinct names (`test_features.py`, `test_data.py`) in bigger projects.
- **The editor recreates a moved file.** `mv test_features.py tests/` on the command line doesn't tell VS Code the file moved — an open tab still points at the *old* path, so the next save **recreates the file there**. Check the editor's title bar shows the path I expect. Close stale tabs after moving files.
- **`.pytest_cache/`** appears once pytest runs — add it to `.gitignore` (it's a generated result). Adding the pattern afterwards is fine *as long as it was never committed*: `.gitignore` only fails to work retroactively on already-**tracked** files.

---

## CI — what and why

**Continuous Integration:** every time I push, a **fresh, clean machine** in GitHub's cloud checks out the code, installs dependencies from the lockfile, and runs the tests.

Two guarantees:
1. The tests **always** run — I can't forget.
2. They run **somewhere that isn't my laptop** — so "works on my machine" stops being the standard. If the project only works because of something installed years ago, CI catches it immediately.

This is what turns a PR from a self-approved formality into a **real gate**: the PR page shows a green tick or red cross *before* the merge button is worth pressing.

**GitHub Actions vocabulary:**

| Term | Means |
|---|---|
| **Workflow** | A YAML file in `.github/workflows/` describing a job |
| **Trigger** (`on:`) | *When* it runs — on push, on pull request, on a schedule |
| **Runner** | The fresh virtual machine GitHub spins up |
| **Job** | A named unit of work that runs on a runner |
| **Step** | One command or action, run in order. **Any step failing fails the whole run** |
| `uses:` | Pull in a pre-built action someone else wrote |
| `run:` | Execute a shell command |

The workflow file is **committed to the repo** — infrastructure defined as code, versioned alongside the code it tests. That idea (config as committed files, not clicks in a UI) carries through to Terraform later.

**YAML:** indentation is **meaningful** — nesting is expressed by spaces, and it must be **spaces, never tabs**. Inconsistent indentation is the most common YAML error.

---

## A working workflow

`.github/workflows/tests.yml`:

```yaml
name: Tests

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5

      - name: Set up Python
        run: uv python install

      - name: Install dependencies
        run: uv sync --all-extras --dev

      - name: Run tests
        run: uv run pytest
```

- `name:` — the label shown on GitHub (appears as the check name on the PR).
- `on:` — triggers. This runs on **pull requests** and on **pushes to `main`**.
- `runs-on: ubuntu-latest` — a fresh **Linux** machine, deliberately not my Mac.
- `actions/checkout@v4` — clones my repo onto the runner.
- `astral-sh/setup-uv@v5` — installs uv.
- Then the commands I already know: install Python, `uv sync` (rebuilds from the lockfile), `uv run pytest`.

**`uv.lock` is what makes this meaningful.** Without it committed, `uv sync` on the runner would resolve `pandas>=3.0.5` fresh and might install a *different* version than I develop against — so a green tick wouldn't prove my code works with my setup.

---

## Two runs per merge: gate vs verification

With both triggers, merging a PR produces **two** runs:
- The **`pull_request`** run — tests the branch *before* it lands. A **gate**.
- The **`push: [main]`** run — tests `main` *after* the merge. A **safety net**.

Not redundant: if someone merged something else in between, a branch can pass in isolation and still break `main` once combined. Gate before, verification after.

**Where to look:** the **Actions tab** is the control panel for all runs. Click a run → expand a step → read its logs. On a PR, the checks box sits below the description (yellow dot = running, green tick = passed, red cross = failed). Runners take 30–60s to spin up.

---

## Pinning the Python version

**The hole:** `uv.lock` pins *packages* but **not the Python interpreter itself**. `requires-python = ">=3.13"` is a range, so the runner installs whatever's newest that satisfies it — CI ran **3.14.7** while local was **3.13.7**. Right now harmless; eventually it produces a green tick that lies.

**The fix:**
```
uv python pin 3.13
```
Writes a `.python-version` file containing the version string. uv reads it automatically — locally when building `.venv`, and on the runner. **Commit it** — it's recipe, not result.

Same pattern as everywhere else: `pyproject.toml` holds the flexible *intent* (`>=3.13`), `.python-version` holds the *exact reality*.

Pinning `3.13` fixes the **series**, not the patch (CI then ran 3.13.15 vs local 3.13.7). That's the right trade: patch releases are bug fixes and security patches within a stable API, so I want them, and pinning to the patch would mean bumping it by hand forever.

> A stray `.python-version` in the **home directory** is a different matter — that's the root-marker footgun (see the Python setup sheet). The file is right; the location was wrong.

---

## What a green tick does and doesn't prove

**Does:** the code works on **Ubuntu Linux** with **my exact locked package versions** and pinned Python series.

**Doesn't:** say anything about macOS, Windows, other Python versions, or newer dependencies — none of those were tested.

That's a deliberate choice, not a flaw. CI should prove the code works **in the environment it will actually deploy to** — and the target is a Linux container on AWS, so Ubuntu is exactly right. (If broader coverage were ever needed, Actions supports a **matrix**: the same job across several OSes and Python versions in parallel. Worth knowing it exists.)

---

## Habits worth keeping

1. Write the test, then **break it deliberately** and watch it fail. Unproven safety nets aren't safety nets.
2. Tiny, obviously-correct inputs; descriptive test names that read as sentences.
3. `uv add --dev` for anything that's a development tool, never plain `uv add`.
4. Read pytest's `E assert <actual> == <expected>` line first — it's usually the whole answer.
5. Read error messages properly: `import file mismatch` printed both conflicting paths; the PAT rejection named the exact missing scope.
6. Never merge a red cross.
