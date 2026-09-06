# Lab Setup

Your working environment is a piece of the curriculum, not a preliminary to it. Set it up
once, properly, and it will carry you through six terms.

## 1. Interpreters

You need **at least three** Python builds available, because several courses depend on
comparing them.

| Build | Why you need it |
|---|---|
| CPython 3.12 | The baseline for all lesson code |
| CPython 3.13 or newer | Free-threaded build experiments (PY-601), JIT observations (PY-602) |
| PyPy 3 | Contrast in PY-602: how much of "Python is slow" is CPython specifically |

Use a version manager rather than the system Python. On Windows, `py` launcher plus
`uv python install` is the least friction; on Linux/macOS, `uv` or `pyenv`.

```bash
# uv is the recommended toolchain throughout this program
uv python install 3.12 3.13
uv python list
```

For the free-threaded build you need the variant explicitly (naming may vary by release —
check `uv python list --all-versions` for a `+freethreaded` or `t`-suffixed build). If you
cannot get one locally, a container works fine:

```bash
docker run -it --rm python:3.13-slim python -VV
```

Verify what you have before PY-601 L03:

```python
import sys, sysconfig
print(sys.version)
print("free-threaded:", not sysconfig.get_config_var("Py_GIL_DISABLED") in (None, 0))
```

## 2. The study repository

Create one repository for the whole program. Not one per course — one. You want the
history to show three years of accumulation.

```
mpse/
├── notes/                 # your notes, one file per lesson
├── log/                   # weekly learning log
├── courses/
│   ├── py501/
│   │   ├── exercises/     # one module per exercise
│   │   └── tests/
│   ├── se511/
│   └── ...
├── artifacts/             # term build artifacts, each its own package
├── papers/                # PDFs of primary sources + your margin notes as .md
└── pyproject.toml
```

Initialize it:

```bash
mkdir mpse && cd mpse
git init
uv init --package
uv add --dev pytest pytest-cov hypothesis mypy ruff
```

Commit after every study session, with the lesson code in the message
(`py501-L04: descriptor exercises`). The log is the evidence.

## 3. Toolchain

Install once, use throughout:

| Tool | Role | Introduced in |
|---|---|---|
| `uv` | Environments, dependency resolution, Python installs | Setup |
| `pytest` | Test runner | SE-511 L01 |
| `hypothesis` | Property-based testing | SE-511 L04 |
| `coverage` / `pytest-cov` | Coverage measurement (and its critique) | SE-511 L03 |
| `mypy` | Static type checking | SE-511 L06 |
| `pyright` | Second checker — they disagree, instructively | SE-511 L07 |
| `ruff` | Lint + format | SE-511 L02 |
| `py-spy` | Sampling profiler | PY-602 L02 |
| `memray` | Memory profiler | PY-602 L03 |
| `scalene` | CPU+memory+GPU profiler | PY-602 L02 |
| `dis`, `sys`, `gc` | Standard library introspection | PY-501 L07 |
| `tla2tools` / TLA+ Toolbox | Model checking | FM-751 L03 |
| `z3-solver` | SMT solving | FM-751 L06 |
| Docker | Multi-node experiments | DS-701 L02 |

```bash
uv add --dev pytest pytest-cov hypothesis mypy pyright ruff py-spy memray scalene z3-solver
```

TLA+ is a separate download (Java-based); FM-751 L03 walks through it.

## 4. Editor configuration

Whatever editor you use, ensure these are live *as you type*:

- Type checking (mypy or pyright language server) — you want the feedback loop tight.
- `ruff` on save.
- A REPL you can send a selection to. Much of PY-501 and PY-502 is best learned by
  poking at objects interactively.

Configure your project once:

```toml
# pyproject.toml
[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "A", "C4", "SIM", "RUF"]

[tool.mypy]
python_version = "3.12"
strict = true
warn_unreachable = true
show_error_codes = true

[tool.pytest.ini_options]
addopts = "-q --strict-markers --strict-config"
testpaths = ["tests"]
```

You will be told in SE-511 *why* each of those settings is there. For now, use them.

## 5. Reading environment

You will read perhaps forty papers over the program. Set up for it:

- A folder of PDFs, named `YYYY-firstauthor-shorttitle.pdf`.
- For each paper, a markdown file with: the claim in one sentence, the method, what you
  did not understand, and one question you would ask the author. Four lines. Do this
  *every time* — it is the difference between having read a paper and having skimmed one.
- `appendices/paper-index.md` lists the required and recommended readings per course.

## 6. Experiment hygiene

Several courses ask you to measure things. Measurements you cannot reproduce are anecdotes.

- Record: machine, OS, Python build, CPU governor / power plan, and whether anything else
  was running.
- Pin the CPU where possible (`taskset` on Linux; on Windows, set affinity).
- Report medians and a spread, never a single run. PY-602 L01 covers the statistics.
- Keep raw data in the repo alongside the analysis script.

## 7. Windows-specific notes

You are on Windows. Most of this program works natively, but three areas do not:

- **`os.fork` and `multiprocessing` start methods** (PY-601 L08): Windows has no `fork`.
  The lessons cover the difference explicitly, but for the exercises that require `fork`
  semantics, use WSL2.
- **`resource`, `signal` coverage, and some `/proc` inspection** (PY-602): use WSL2.
- **Container-based distributed experiments** (DS-701, DI-721): Docker Desktop with the
  WSL2 backend.

Install WSL2 with a Debian or Ubuntu image and keep a parallel `mpse` checkout there. The
lessons flag which exercises need it.

## 8. A five-minute check that you're ready

```bash
uv run python -c "import sys; print(sys.version_info)"
uv run pytest --version
uv run mypy --version
uv run ruff --version
uv run python -c "import hypothesis, z3; print('ok')"
docker run --rm hello-world
git log --oneline | head -1
```

If all seven print something sensible, begin `term-1/PY-501-python-object-model/syllabus.md`.
