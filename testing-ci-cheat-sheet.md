# Testing & CI Cheat Sheet — pytest & GitHub Actions

---

## Why

```
  WITHOUT tests                      WITH tests
  run it, squint at the output       uv run pytest
  ~30 seconds, every time            0.28 seconds
  might forget to check              checks everything, always
  "was 215 right?" (6 months on)     the answer is written down
```

The quieter benefit: tests make you **willing to change code**. Without them refactoring is nerve-wracking.

**A test you've never seen fail is unproven.** Break it on purpose, watch it fail, fix it. Same for CI.

---

## A test is three steps

```
  1. SET UP     a known input (tiny, obviously correct)
       │
       ▼
  2. RUN        my real code on it
       │
       ▼
  3. ASSERT     result == what I expect
```

`assert` = "this must be true, or fail loudly". That's the whole idea.

---

## Setup

```
  uv add --dev pytest
```

The `--dev` matters:

```
  pyproject.toml
  ┌──────────────────────────┐
  │ [project]                │
  │ dependencies = [         │
  │   "pandas",              │ ──► SHIPS in the Docker image
  │   "fastapi",             │
  │ ]                        │
  │                          │
  │ [dependency-groups]      │
  │ dev = [                  │
  │   "pytest",              │ ──► does NOT ship
  │   "httpx",               │     (Dockerfile runs --no-dev)
  │ ]                        │
  └──────────────────────────┘
```

**Where tests live** — outside `src/`, because they don't ship:

```
  project/
  ├── src/<package>/      ← ships
  └── tests/              ← does NOT ship
      ├── test_features.py
      └── test_api.py
```

**Discovery is by naming** — pytest only looks for:
```
  files:      test_*.py
  functions:  test_*
```
Miss the prefix → **silently ignored**.

**Tests import absolutely** (they're outside the package, so no relative imports):
```python
from santander_cycles_forecast.features import total_by_station
```

---

## A test

```python
import pandas as pd
from santander_cycles_forecast.features import total_by_station


def test_total_by_station_sums_hires_per_station():
    df = pd.DataFrame({                              # 1. SET UP
        "station": ["Hyde Park", "Waterloo", "Hyde Park"],
        "hires":   [100, 50, 20],
    })

    result = total_by_station(df)                    # 2. RUN

    assert result.loc[result["station"] == "Hyde Park", "hires"].iloc[0] == 120
    assert len(result) == 2                          # 3. ASSERT
```

Name it as a **sentence** — it's what shows up when it fails.

That assertion unpacked:
```
  result["station"] == "Hyde Park"     → column of True/False
  result.loc[that, "hires"]            → the hires value where True  (SQL's WHERE)
  .iloc[0]                             → first one, by position
```

---

## Reading pytest output

```
  uv run pytest          ← no file argument; it finds them
```

**Pass:**
```
  collected 3 items

  tests/test_api.py ..                [ 66%]
  tests/test_features.py .            [100%]
                        ▲
                        one character per test
                        . = pass   F = fail   E = error

  3 passed in 0.31s
```

**Fail — pytest shows you everything:**
```
  tests/test_features.py F            [100%]

      def test_total_by_station_sums_hires_per_station():
          df = pd.DataFrame({...})
          result = total_by_station(df)
  >       assert result.loc[...].iloc[0] == 120
  E       assert np.int64(120) == 999
                       ▲         ▲
                       ACTUAL    EXPECTED

  tests/test_features.py:16: AssertionError
```

That `E` line is the gift — in real debugging you don't know the actual value. **Read it first.**

**Type tells:** `.sum()` gives `int64`, `.mean()` gives `float64`. A stray float hints at what went wrong.

---

## pytest gotchas

| Problem | Why |
|---|---|
| `import file mismatch` | Two `test_features.py` in different folders. pytest imports by module name → collision. The error prints **both paths** — read them |
| A moved file comes back | VS Code still has the **old path** open. Saving recreates it. Check the title bar; close stale tabs |
| `.pytest_cache/` appears | Generated — add to `.gitignore` (fine to add after, as long as it was never committed) |

---

## CI — what it buys you

```
  WITHOUT CI                      WITH CI
  tests run when I remember       tests run on EVERY push & PR
  only on my Mac                  on a fresh Ubuntu machine
  "works on my machine"           proof it works elsewhere
```

If the project only works because of something you installed years ago, CI catches it immediately.

**It turns a PR into a real gate:**

```
  my branch ──► PR ──► [ Tests ✅ ] ──► merge allowed
                       [ Tests ❌ ] ──► don't merge
```

---

## Vocabulary

| Term | Means |
|---|---|
| **Workflow** | A YAML file in `.github/workflows/` |
| **Trigger** (`on:`) | *When* it runs |
| **Runner** | The fresh VM GitHub spins up |
| **Step** | One command. **Any step failing fails the run** |
| `uses:` | Use a pre-built action someone wrote |
| `run:` | Execute a shell command |

The workflow is **committed to the repo** — infrastructure as code, same idea as a Dockerfile. Carries through to Terraform later.

**YAML: indentation is meaningful. Spaces only, never tabs.**

---

## The workflow

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

What happens on the runner:

```
  ┌─ fresh Ubuntu VM, nothing installed ─────────────┐
  │                                                  │
  │  checkout    → clone my repo                     │
  │  setup-uv    → install uv                        │
  │  python      → install Python (reads             │
  │                .python-version)                  │
  │  uv sync     → install EXACT versions            │
  │                from uv.lock       ◄── the key    │
  │  pytest      → run the tests                     │
  │                                                  │
  └──────────────────────────────────────────────────┘
```

**Without `uv.lock` committed**, the runner would resolve `pandas>=3.0.5` fresh and might install something different — so a green tick wouldn't prove anything about *my* setup.

---

## Two runs per merge

```
  open PR  ──► pull_request run   ← GATE: test the branch before it lands
                     │
                  merge
                     │
                     ▼
           push:[main] run        ← SAFETY NET: test main after
```

Not redundant — if someone else merged in between, a branch can pass alone and still break `main` combined.

**Where to look:** the **Actions tab** lists every run. Click a run → expand a step → read logs. On a PR, the checks box sits under the description (yellow = running, green = passed, red = failed). Runners take 30–60s to start.

---

## What a green tick proves

```
  ✅ proves:        Ubuntu Linux + my exact locked packages + pinned Python
  ❌ says nothing:  macOS, Windows, other Python versions, newer deps
```

That's deliberate, not a gap. CI should test **the environment you deploy to** — Linux containers on AWS. (A **matrix** can run several OSes/versions in parallel if ever needed.)

---

## Gotchas

| Error | Fix |
|---|---|
| `refusing to allow a Personal Access Token to create or update workflow ... without 'workflow' scope` | Pushing `.github/workflows/*.yml` needs the **`workflow`** scope, separate from `repo`. It's the **403 family** — authenticated but not authorised. Workflow files are *code GitHub runs on its own servers*, so a stolen token shouldn't be able to add one. Tick `workflow` on the token, clear the stale Keychain entry |
| CI passes locally, fails on runner | Usually a missing dev dependency (e.g. `httpx` for FastAPI's TestClient) |
| CI Python ≠ local Python | `uv.lock` doesn't pin the interpreter. `uv python pin 3.13` → commit `.python-version` |

---

## Habits

1. Write it, **break it**, watch it fail, fix it
2. Tiny inputs; names that read as sentences
3. `uv add --dev` for anything development-only
4. Read pytest's `E assert <actual> == <expected>` line first
5. Never merge a red cross
