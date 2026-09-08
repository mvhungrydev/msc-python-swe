# Lab Setup — Step by Step

Your working environment is part of the curriculum, not a preliminary to it. Set it up once,
properly, and it carries you through six terms.

This guide has **two complete tracks**. Follow the one for your machine, top to bottom.
Every step ends with a check you can run; do not move on until the check passes.

- [Track A — Windows 11 / 10](#track-a--windows)
- [Track B — macOS (Apple Silicon or Intel)](#track-b--macos)
- [Part 3 — Common setup (both platforms)](#part-3--common-setup-both-platforms)
- [Part 4 — Verification](#part-4--verification)
- [Part 5 — Troubleshooting](#part-5--troubleshooting)

**Time:** about 60–90 minutes including downloads.

---

## What you are installing, and why

| Tool | Why the program needs it | First used |
|---|---|---|
| `uv` | Python versions, virtualenvs, dependency resolution, lockfiles | Setup |
| CPython 3.12 | Baseline for all lesson code | PY-501 L01 |
| CPython 3.13+ | Free-threaded build experiments; JIT observations | PY-601 L02 |
| PyPy 3 | Contrast: how much of "Python is slow" is CPython specifically | PY-602 L01 |
| Git | The study repository; change-coupling analysis | Setup |
| Docker | Multi-node experiments, real databases in tests | SE-511 L05 |
| WSL2 (Windows only) | `fork`, `resource`, `/proc`, native container performance | PY-601 L08 |
| Java 17+ | TLA+ model checker | FM-751 L03 |
| A C toolchain | Building extensions; Cython experiments | PY-602 L07 |

---

# Track A — Windows

Tested on Windows 11. Windows 10 (21H2+) works identically.

You will end up with **two environments**: native Windows (your day-to-day) and **WSL2**
(for the ~15% of exercises that need real Unix). This is deliberate — the program tells you
which exercises need which.

## A1. Open an elevated PowerShell

Press `Win`, type `PowerShell`, right-click **Windows PowerShell** → **Run as administrator**.

Allow local scripts for this user (needed by several installers):

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

**Check:** `Get-ExecutionPolicy -Scope CurrentUser` prints `RemoteSigned`.

## A2. Install a package manager (winget)

`winget` ships with Windows 11 and recent Windows 10. Check:

```powershell
winget --version
```

If it is missing, install **App Installer** from the Microsoft Store, then reopen PowerShell.

## A3. Install Git

```powershell
winget install --id Git.Git -e --source winget
```

Then close and reopen PowerShell (so `PATH` updates) and configure it:

```powershell
git config --global user.name  "Mike Velasco"
git config --global user.email "mikevelasco16@gmail.com"
git config --global init.defaultBranch main
git config --global core.autocrlf true
git config --global pull.rebase true
```

**Check:** `git --version` prints 2.4x or newer.

> `core.autocrlf true` on Windows checks out CRLF and commits LF. Leave it on unless you
> know you need otherwise — it prevents a whole class of noisy diffs.

## A4. Install Windows Terminal (recommended)

```powershell
winget install --id Microsoft.WindowsTerminal -e
```

Gives you tabs, a decent font, and one place to run PowerShell and WSL side by side.

## A5. Install the C build tools

Needed to compile Python extensions (PY-602 L07) and any wheel with no prebuilt Windows
binary.

```powershell
winget install --id Microsoft.VisualStudio.2022.BuildTools -e `
  --override "--quiet --wait --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"
```

A ~2 GB download; several minutes. It installs the MSVC compiler and the Windows SDK, with
no full Visual Studio IDE.

**Check:** open a *new* PowerShell and run

```powershell
& "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" -latest -products * `
  -requires Microsoft.VisualStudio.Component.VC.Tools.x86.x64 -property installationPath
```

It should print a path. If it prints nothing, the workload did not install — rerun A5.

## A6. Install `uv`

`uv` is the toolchain for the whole program: it installs Python versions, creates
environments, resolves dependencies, and writes lockfiles.

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Close and reopen PowerShell.

**Check:** `uv --version` prints something. If "not recognized", see
[T1](#t1-uv-not-recognized).

## A7. Install the Python interpreters

```powershell
uv python install 3.12
uv python install 3.13
uv python list
```

Then the free-threaded build (needed for PY-601 L02):

```powershell
uv python list --all-versions | Select-String "freethreaded"
uv python install 3.13t          # 't' suffix = free-threaded; match what the list shows
```

If no free-threaded build is offered for your platform, skip it — you will use Docker for
those two experiments instead (see A10).

**Check:**

```powershell
uv python list
uv run --python 3.12 python -c "import sys; print(sys.version)"
```

## A8. Install PyPy

Used only in PY-602 for comparison, so this can wait until Term 3.

```powershell
uv python install pypy@3.10
```

If that fails, download from [pypy.org/download.html](https://pypy.org/download.html), unzip
to `C:\pypy3`, and add that folder to your PATH.

## A9. Install WSL2

About 15% of the program's exercises need genuine Unix behaviour: `os.fork` and the
`multiprocessing` fork start method (PY-601 L08), the `resource` module, `/proc` inspection,
full `signal` coverage, and `perf`/eBPF tooling (PY-602).

```powershell
wsl --install -d Ubuntu
```

Reboot when prompted. On first launch it asks for a UNIX username and password — these are
independent of your Windows account.

**Check:**

```powershell
wsl --status          # should say "Default Version: 2"
wsl -l -v             # Ubuntu, VERSION 2
```

If it says version 1: `wsl --set-version Ubuntu 2`.

Now set up the Linux side. Open Ubuntu (from Start, or type `wsl` in PowerShell):

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential git curl unzip pkg-config \
                    linux-tools-common linux-tools-generic \
                    default-jre graphviz
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc
uv python install 3.12 3.13
```

**Check (inside WSL):** `uv --version` and `uv run --python 3.12 python -VV` both work.

> **Where to keep your files.** Keep the study repo **inside** the WSL filesystem
> (`~/mpse`) when working in Linux, and a separate checkout on Windows (`C:\dev\mpse`).
> Working on `/mnt/c/...` from WSL is 5–20× slower for git and file-heavy operations. Two
> checkouts of the same remote is the least painful arrangement.

## A10. Install Docker Desktop

```powershell
winget install --id Docker.DockerDesktop -e
```

Launch Docker Desktop, then check **Settings → General → Use the WSL 2 based engine** (on by
default) and **Settings → Resources → WSL Integration → enable for Ubuntu**.

**Check** — both must work:

```powershell
docker run --rm hello-world
```

```bash
# inside WSL
docker run --rm hello-world
```

With Docker you can get a free-threaded interpreter even if `uv` has no build for you:

```powershell
docker run -it --rm python:3.13-slim python -VV
```

## A11. Install Java (for TLA+)

Needed in Term 6, FM-751. Install now or later.

```powershell
winget install --id EclipseAdoptium.Temurin.21.JDK -e
```

**Check:** `java -version` in a new shell.

Windows track complete → continue to [Part 3](#part-3--common-setup-both-platforms).

---

# Track B — macOS

Tested on macOS 14/15, Apple Silicon and Intel.

## B1. Install the Command Line Tools

```bash
xcode-select --install
```

A dialog appears; accept it. This gives you `git`, `clang`, `make`, and the SDK headers —
required to build any Python extension.

**Check:** `xcode-select -p` prints a path, and `clang --version` works.

## B2. Install Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

The installer prints two commands at the end to put Homebrew on your PATH. **Run them.** On
Apple Silicon they are:

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

On Intel Macs the prefix is `/usr/local` instead.

**Check:** `brew --version` works and `which brew` prints `/opt/homebrew/bin/brew`
(or `/usr/local/bin/brew` on Intel).

## B3. Configure Git

macOS ships git, but Homebrew's is newer:

```bash
brew install git
git config --global user.name  "Mike Velasco"
git config --global user.email "mikevelasco16@gmail.com"
git config --global init.defaultBranch main
git config --global core.autocrlf input
git config --global pull.rebase true
```

**Check:** `git --version` prints 2.4x or newer.

## B4. Install `uv`

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Restart your shell, or `source ~/.zshrc`.

**Check:** `uv --version`. If "command not found", see [T1](#t1-uv-not-recognized).

## B5. Install the Python interpreters

```bash
uv python install 3.12
uv python install 3.13
uv python list
```

The free-threaded build (PY-601 L02):

```bash
uv python list --all-versions | grep -i freethreaded
uv python install 3.13t          # match whatever the list shows
```

**Check:**

```bash
uv run --python 3.12 python -c "import sys; print(sys.version)"
```

> **Do not use the system Python** (`/usr/bin/python3`). It belongs to macOS, the OS uses
> it, and installing into it is a mistake you have to undo later. Everything in this program
> goes through `uv`.

## B6. Install PyPy

```bash
uv python install pypy@3.10
```

or `brew install pypy3` as a fallback. Not needed until Term 3.

## B7. Install Docker

```bash
brew install --cask docker
open -a Docker
```

Accept the privileged-helper prompt on first launch.

**Check:**

```bash
docker run --rm hello-world
```

**Apple Silicon note:** some images are `amd64`-only and run under emulation, which is slow
and occasionally subtly wrong. Prefer `arm64` images; when you must use an `amd64` one, pass
`--platform linux/amd64` explicitly so the emulation is visible rather than silent.

## B8. Install supporting tools

```bash
brew install graphviz jq watch coreutils
brew install --cask temurin@21      # Java, for TLA+ in FM-751
```

`coreutils` gives you GNU `ls`, `date`, `timeout` and friends as `g`-prefixed commands
(`gdate`, `gtimeout`). Several lessons assume GNU behaviour; this saves confusion.

**Check:** `dot -V`, `jq --version`, and `java -version` all work.

## B9. Profiling permissions

`py-spy` and `scalene` read another process's memory, which macOS restricts.

```bash
sudo DevToolsSecurity -enable
```

You will also be prompted the first time; grant it. If `py-spy` reports a permissions error,
run it under `sudo`.

macOS track complete → continue to Part 3.

---

# Part 3 — Common setup (both platforms)

From here the commands are identical on Windows PowerShell, macOS, and WSL.

## 3.1 Create the study repository

**One repository for the whole program.** Not one per course — you want three years of
history in one place.

```bash
mkdir mpse
cd mpse
git init
uv init --package --name mpse
mkdir -p notes log courses artifacts papers/notes scripts
```

Final shape:

```
mpse/
├── notes/          one file per lesson — your own words, written from memory
├── log/            weekly learning log; log/failures.md
├── courses/        exercises and problem sets, one folder per course
├── artifacts/      term build artifacts, each its own package
├── papers/         PDFs of primary sources + a four-line note per paper
├── scripts/        your own tooling (linters, analyzers) as you build it
└── pyproject.toml
```

## 3.2 Install the toolchain

```bash
uv add --dev pytest pytest-cov pytest-randomly hypothesis mypy ruff pyright
uv add --dev py-spy scalene z3-solver
uv add --dev memray            # Linux/macOS only — skip on native Windows, use WSL
```

**Check:**

```bash
uv run pytest --version
uv run mypy --version
uv run ruff --version
uv run python -c "import hypothesis, z3; print('ok')"
```

## 3.3 Configure the project

Replace `pyproject.toml` with:

```toml
[project]
name = "mpse"
version = "0.1.0"
description = "MSc-equivalent program in Advanced Python & Software Engineering"
requires-python = ">=3.12"
dependencies = []

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = ["E", "W", "F", "I", "N", "UP", "B", "A", "C4", "SIM", "DTZ", "PTH", "RUF"]

[tool.mypy]
python_version = "3.12"
strict = true
warn_unreachable = true
show_error_codes = true
pretty = true

[tool.pytest.ini_options]
addopts = "-q --strict-markers --strict-config"
testpaths = ["courses", "artifacts"]
markers = [
    "small: no I/O, no sleep, no network",
    "medium: localhost, filesystem, containers",
    "large: multiple machines or real external services",
    "slow: excluded from the default run",
]

[tool.coverage.run]
branch = true

[tool.coverage.report]
exclude_also = ["if TYPE_CHECKING:", "raise NotImplementedError", "@overload"]
```

SE-511 explains *why* each of these settings is there. For now, use them.

## 3.4 Pin the interpreter and lock

```bash
uv python pin 3.12
uv sync
git add -A
git commit -m "lab: initial environment"
```

**Check:** `uv sync --frozen` succeeds. (It fails if the lock is stale — this is what CI will
run.)

## 3.5 Editor setup

Whatever editor you use, make sure these are live **as you type**:

- Type checking (the mypy or pyright language server)
- `ruff` on save
- A REPL you can send a selection to — much of PY-501 and PY-502 is learned by poking at
  objects interactively

For VS Code:

```bash
code --install-extension ms-python.python
code --install-extension ms-python.vscode-pylance
code --install-extension charliermarsh.ruff
```

Then `.vscode/settings.json`:

```json
{
  "python.defaultInterpreterPath": ".venv/bin/python",
  "python.analysis.typeCheckingMode": "strict",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": { "source.organizeImports.ruff": "explicit" },
  "[python]": { "editor.defaultFormatter": "charliermarsh.ruff" }
}
```

On Windows the interpreter path is `.venv\\Scripts\\python.exe`.

## 3.6 Reading environment

You will read about forty papers. Set up for it now:

- PDFs named `YYYY-firstauthor-shorttitle.pdf` in `papers/`.
- For every paper, a file in `papers/notes/` with four lines: the claim in one sentence, the
  method, what you did not understand, and one question you would ask the author.

Four lines. Do it *every time* — it is the difference between having read a paper and having
skimmed one. `00-program/reading-list.md` lists the required and recommended readings per
course.

## 3.7 Experiment hygiene

Several courses ask you to measure things. Measurements you cannot reproduce are anecdotes.

Create `scripts/envinfo.py` and run it at the top of every measurement writeup:

```python
"""Print the environment facts every measurement must record."""
import json, os, platform, subprocess, sys, sysconfig


def main() -> None:
    info = {
        "python": sys.version,
        "implementation": platform.python_implementation(),
        "gil_disabled": bool(sysconfig.get_config_var("Py_GIL_DISABLED")),
        "platform": platform.platform(),
        "machine": platform.machine(),
        "processor": platform.processor(),
        "cpu_count": os.cpu_count(),
    }
    try:
        info["git_rev"] = subprocess.check_output(
            ["git", "rev-parse", "--short", "HEAD"], text=True
        ).strip()
    except Exception:
        info["git_rev"] = None
    print(json.dumps(info, indent=2))


if __name__ == "__main__":
    main()
```

```bash
uv run python scripts/envinfo.py
```

Also record by hand: power plan / CPU governor, whether anything else was running, and
whether the machine was on battery. Report **medians and a spread, never a single run** —
PY-602 L01 covers the statistics.

Pin the CPU where possible:

- **Linux / WSL:** `taskset -c 2 uv run python bench.py`
- **Windows:** `start /affinity 4 cmd /c "uv run python bench.py"`
- **macOS:** no direct equivalent. Close everything else; use `sudo powermetrics` to confirm
  no background load. Also disable Low Power Mode and keep the machine on mains power —
  thermal and power state affect results more on laptops than most people expect.

## 3.8 Commit discipline

Commit after every study session, with the lesson in the message:

```bash
git add -A
git commit -m "py501-L04: descriptor exercises + MRO by hand"
```

The log is the evidence. When motivation sags in month fourteen,
`git log --oneline | wc -l` is what shows you the distance travelled.

---

# Part 4 — Verification

Run all of these. Every one must produce sensible output before you start PY-501.

```bash
uv run python -c "import sys; print(sys.version_info)"
uv run python scripts/envinfo.py
uv run pytest --version
uv run mypy --version
uv run ruff --version
uv run pyright --version
uv run python -c "import hypothesis, z3; print('hypothesis + z3 ok')"
uv run py-spy --version
uv python list
docker run --rm hello-world
git log --oneline | head -1
java -version
```

**Windows** — also verify WSL:

```powershell
wsl -e bash -lc "uv --version && uv run --python 3.12 python -c 'import os; print(os.fork)'"
```

That last one confirms `fork` exists in WSL (it does not on native Windows), which is what
PY-601 L08 needs.

**macOS** — verify the compiler:

```bash
echo 'int main(){return 0;}' > /tmp/t.c && clang /tmp/t.c -o /tmp/t && echo "clang ok"
```

**Both** — verify you can build an extension, which several PY-602 exercises need:

```bash
uv pip install cython
printf 'def f(int n):\n    cdef int i, s = 0\n    for i in range(n): s += i\n    return s\n' > m.pyx
uv run cythonize -i m.pyx
uv run python -c "import m; print(m.f(1000))"
```

If that prints `499500`, your toolchain is complete.

**Free-threading check** (PY-601 L02 needs one of these to work):

```bash
uv run --python 3.13t python -c "import sysconfig; print('free-threaded:', bool(sysconfig.get_config_var('Py_GIL_DISABLED')))"
# or, with no local free-threaded build:
docker run --rm python:3.13-slim python -c "import sysconfig; print(sysconfig.get_config_var('Py_GIL_DISABLED'))"
```

When all of this passes, begin
[`term-1/PY-501-python-object-model/syllabus.md`](../term-1/PY-501-python-object-model/syllabus.md).

---

# Part 5 — Troubleshooting

### T1. `uv` not recognized

The installer added `uv` to a directory that is not on your PATH in the *current* shell.

- **Windows:** it installs to `%USERPROFILE%\.local\bin`. Close and reopen PowerShell. If
  still missing:
  ```powershell
  $env:Path += ";$env:USERPROFILE\.local\bin"
  [Environment]::SetEnvironmentVariable("Path", $env:Path, "User")
  ```
- **macOS:** it installs to `~/.local/bin`:
  ```bash
  echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
  ```

### T2. `uv python install` fails to download

Corporate proxy or TLS interception. Set `SSL_CERT_FILE` to your organization's CA bundle,
or `UV_PYTHON_INSTALL_MIRROR` if you have an internal mirror. As a fallback, install Python
from python.org and point `uv` at it: `uv venv --python C:\Path\To\python.exe`.

### T3. A package fails to build a wheel on Windows

Almost always the missing C toolchain — redo [A5](#a5-install-the-c-build-tools) and make
sure you opened a *new* shell afterwards. If it still fails, the package has no Windows wheel
at all; build it in WSL instead.

### T4. `memray` will not install

Linux/macOS only. On Windows, run the PY-602 memory exercises inside WSL. The lessons flag
which those are.

### T5. `py-spy` permission denied on macOS

```bash
sudo DevToolsSecurity -enable
sudo py-spy top --pid <pid>
```

macOS restricts reading another process's memory; running the profiler under `sudo` is the
supported path.

### T6. Docker on Windows says "WSL 2 is not installed"

`wsl --install`, reboot, then `wsl --set-default-version 2`. Then in Docker Desktop:
**Settings → Resources → WSL Integration**, and enable your distro.

### T7. WSL filesystem is slow

You are working under `/mnt/c/...`. Move the repo into the WSL filesystem (`~/mpse`) and keep
a separate Windows checkout. Cross-filesystem access in WSL2 goes over a network protocol and
is 5–20× slower for git operations.

### T8. Apple Silicon: a package has no arm64 wheel

```bash
docker run --platform linux/amd64 -it --rm python:3.12 bash
```

Prefer running that one exercise in a container over installing an x86 Python and creating
two parallel worlds on your Mac.

### T9. `uv sync --frozen` fails

The lockfile is out of date relative to `pyproject.toml`. Run `uv lock`, review the diff
(SE-511 L09 makes reviewing lockfile diffs a habit worth having), and commit.

### T10. `mypy --strict` reports hundreds of errors

Expected on any pre-existing code. SE-511 L06 §2.6 gives the per-module migration strategy;
do not try to fix it all at once. On the empty study repo it should report nothing.

---

## Appendix — one-page cheat sheet

```bash
uv python list                      # what interpreters are available
uv python install 3.13              # install one
uv python pin 3.12                  # set this project's interpreter
uv venv                             # create .venv
uv add httpx                        # add a runtime dependency + update the lock
uv add --dev pytest                 # add a dev dependency
uv sync                             # make the env match the lock
uv sync --frozen                    # fail if the lock is stale (use this in CI)
uv lock --upgrade                   # re-resolve everything
uv run python x.py                  # run inside the project env
uv run --python 3.13 python x.py    # run under a different interpreter
uv export --format requirements-txt # interop with pip-based tools
uv build                            # build sdist + wheel
```
