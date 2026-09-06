# SE-511 · Lesson 09 — Packaging, Dependencies, and Reproducibility

**Estimated study time:** 4 hours
**Prerequisites:** none, though L08 helps

---

## 1. Orientation

"It works on my machine" is not a joke about carelessness. It is a statement about an
*unspecified* environment: a set of package versions, transitive dependencies, a Python
build, a C library, an architecture, and a set of environment variables that were never
written down. Reproducibility is the discipline of making that set explicit.

It matters more than it used to, for two reasons. First, dependency trees have grown: a
modest web service pulls 80–200 packages, almost none of which anyone has read. Second,
that tree is an *attack surface* — `event-stream`, `ua-parser-js`, `xz-utils`, and the
steady stream of typosquatted PyPI packages are not exotic; they are the normal
threat model now.

## 2. Theory

### 2.1 The modern packaging stack

The old world (`setup.py`, `easy_install`, implicit build steps) is gone. The current stack
is defined by PEPs and is genuinely coherent:

| PEP | What it defines |
|---|---|
| 517 / 518 | Build backends and `[build-system]`; `setup.py` is no longer required |
| 621 | Project metadata in `[project]` in `pyproject.toml` |
| 440 | Version specifiers and ordering |
| 508 | Dependency specification (markers: `; python_version < "3.12"`) |
| 427 / 491 | The wheel format |
| 561 | `py.typed` for type information distribution |
| 735 | Dependency groups (dev/test/docs, standardized) |

A complete, modern `pyproject.toml`:

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "mypkg"
version = "0.3.1"
description = "..."
requires-python = ">=3.12"
dependencies = [
    "httpx>=0.27,<0.29",
    "pydantic>=2.7,<3",
]

[dependency-groups]        # PEP 735
dev = ["pytest>=8", "mypy>=1.10", "ruff>=0.5", "hypothesis>=6"]

[project.scripts]
mypkg = "mypkg.__main__:main"
```

**sdist vs wheel.** An sdist is a source archive that must be *built*; a wheel is a
prebuilt, installable artifact. Publish both: the wheel for fast, reliable installs, the
sdist so that people on unusual platforms (and Linux distributions) can build. For pure
Python, one wheel (`py3-none-any`) suffices. For extensions, one wheel per platform/ABI —
`cibuildwheel` is the standard answer and saves an enormous amount of pain.

### 2.2 Version specifiers and what they promise

`>=1.2,<2.0` and `~=1.2` (compatible release: `>=1.2,<2.0`) and `==1.2.*` all encode a
*bet on semantic versioning*, which is a social convention that upstream may or may not
honour.

The honest framing:

- **Applications** should pin exactly, via a lockfile. You control the deployment; take the
  determinism.
- **Libraries** should specify the widest range they actually support, because a narrow
  range in a library propagates and creates unsatisfiable resolutions for consumers.
  A library that pins `httpx==0.27.2` is unusable alongside anything else that depends on
  `httpx`.
- **Upper bounds in libraries are contentious.** Capping `<3` on a dependency prevents your
  users from adopting a compatible `3.0` until you cut a release. Not capping means a
  breaking upstream release breaks your users with no warning. The defensible middle:
  cap only where the dependency has a history of breaking changes, and automate the bump.

Read PEP 440's ordering rules once. Pre-releases, post-releases, epochs, and local versions
(`+cu121`) all have specified ordering and it is not always what you would guess.

### 2.3 Resolution

A resolver must find one version of each package satisfying every constraint. This is
**NP-complete** in general (it encodes SAT — CS-621 L09 will make this precise), which is
why resolution is sometimes slow and why "backtracking" appears in your logs.

Python's model makes it harder than it needs to be: a single flat environment where each
package has exactly one version. Node's nested `node_modules` sidesteps conflicts by
allowing multiple versions; Python cannot. Consequences you will meet:

- **Diamond conflicts** are unresolvable, not merely awkward. If A needs `lib<2` and B
  needs `lib>=2`, you cannot have both. Your options are: fork, vendor, or drop one.
- **Metadata is unreliable.** Some sdists declare dependencies only at build time, so the
  resolver may have to download and build to learn them.
- **Environment markers** mean the resolution is per-platform. A lock produced on macOS may
  not install on Linux unless the tool resolves for multiple platforms (`uv` and `poetry`
  do; naive `pip freeze` does not).

`uv` is the current best answer: a fast resolver, a universal (multi-platform) lockfile,
and Python version management in one tool.

```bash
uv sync                 # create/update the env from uv.lock
uv add httpx            # add a dependency and update the lock
uv lock --upgrade       # re-resolve
uv export --format requirements-txt --no-hashes  # interop
```

### 2.4 Locking, and what it does and does not guarantee

A lockfile records the exact resolved version **and hash** of every package, direct and
transitive.

What it guarantees: the same *distribution artifacts* are installed. That is a lot, and it
is the single highest-value practice in this lesson.

What it does **not** guarantee:

- **The Python interpreter version and build.** Pin it (`.python-version`, `requires-python`,
  a container base image digest).
- **System libraries.** A wheel linking `libssl` behaves differently across base images.
- **Compilation.** Any sdist built at install time compiles against whatever is present.
- **Non-determinism inside the build.** Timestamps, embedded paths, and file ordering make
  even the same source produce different bytes. The *Reproducible Builds* project exists for
  exactly this; `SOURCE_DATE_EPOCH` is the standard mitigation.
- **Anything at runtime.** Environment variables, DNS, feature flags, clock.

The real reproducibility ladder, and where most teams should aim:

1. Lockfile with hashes. **(Do this. It is cheap and it is most of the benefit.)**
2. Pinned interpreter version.
3. Container image pinned by **digest**, not tag. (`python:3.12-slim` moves; the digest does
   not.)
4. Wheels only, no sdists at install time (`--only-binary=:all:` where possible).
5. Bit-for-bit reproducible builds. Expensive; justified for signing and for
   security-critical supply chains.

### 2.5 Supply chain

Threats, concretely:

- **Typosquatting** — `reqeusts`, `python-dateutil` vs `dateutil`. Mitigated by locking and
  by reviewing every *new* direct dependency.
- **Dependency confusion** — an internal package name that also exists on PyPI, and your
  index configuration prefers the public one. Mitigated by using a single proxying index,
  never `--extra-index-url` with a public index alongside a private one (the semantics are
  genuinely dangerous: pip may pick the highest version across *both*).
- **Compromised maintainer account** — a legitimate package publishes a malicious version.
  Mitigated by locking with hashes plus a delay before upgrading (a "cooling-off" period of
  a few days catches most incidents).
- **Malicious install-time code** — `setup.py` runs arbitrary code at install. Prefer
  wheels; they do not execute code at install time (only at import).
- **Transitive additions** — a dependency adds a new dependency in a patch release.
  Visible only if you diff the lockfile.

Practices, in order of value per unit effort:

1. **Lock with hashes.** Everything else is secondary.
2. **Review lockfile diffs** in PRs. Make added packages visible.
3. **`pip-audit` / `osv-scanner`** in CI against known-vulnerability databases.
4. **A single internal index** that proxies and caches PyPI. This also protects you from
   upstream deletions (`left-pad`).
5. **SBOM generation** (CycloneDX or SPDX) if you ship software to others; increasingly a
   procurement requirement.
6. **Trusted publishing** (OIDC-based PyPI publishing from CI) instead of long-lived API
   tokens. This is a genuine improvement and takes twenty minutes to set up.
7. **Minimize.** The strongest control is fewer dependencies. A 40-line vendored function
   is often better than a package with 12 transitive dependencies. Ask, for every new
   dependency: what does it do, how much of it do I use, who maintains it, and what happens
   if it is abandoned?

### 2.6 Publishing

```bash
uv build                                # produces dist/*.whl and dist/*.tar.gz
uv publish --index testpypi             # rehearse on TestPyPI first, always
uv publish
```

A package that is *ready* has: a `README` rendered correctly on PyPI (check on TestPyPI
first — bad markup is the most common release embarrassment), a `LICENSE` and a
`license` field, classifiers, a `project.urls` table with source and issue links,
`requires-python`, a `CHANGELOG`, `py.typed` if typed, and a tag in git matching the
version.

**Versioning.** SemVer is a promise about *your* API, and the hard part is deciding what
counts as your API. Write it down: "public API is everything documented in the reference;
anything under `_internal` may change in any release." Without that sentence every bug fix
is arguably breaking.

## 3. Construction: making a project reproducible

**Step 1 — audit the current state.** For a project you work on, answer: can a new
developer, or CI, get bit-identical dependencies as production? Usually the honest answer
is no, and the reasons are informative.

**Step 2 — introduce a lockfile with hashes.**

```bash
uv lock
uv sync --frozen        # fails if the lock is out of date — use this in CI
```

Commit `uv.lock`. Add a CI job that runs `uv sync --frozen` and fails if the lock does not
match `pyproject.toml`.

**Step 3 — pin the interpreter.** `.python-version`, `requires-python`, and — if
containerized — a base image digest:

```dockerfile
FROM python:3.12-slim@sha256:<digest>
```

**Step 4 — separate build and runtime.** A multi-stage build where the final image contains
no compiler, no build dependencies, and no package manager. Report the size difference; it
is usually 3–5×, and the attack surface reduction is larger than the size reduction.

**Step 5 — measure.** Build the image twice, an hour apart, and compare digests. They will
differ. Identify why (timestamps, `apt` metadata, pip cache). Fix what is cheap to fix
(`SOURCE_DATE_EPOCH`, `--no-cache-dir`, pinned `apt` versions) and *document* the rest.
Documenting the residual non-determinism is the honest deliverable; achieving bit-identical
builds is a project in itself.

**Step 6 — supply chain controls.** Add `pip-audit` to CI. Add a lockfile-diff comment to
PRs. Generate an SBOM. Set up trusted publishing.

## 4. Failure modes

- **`pip install -r requirements.txt` with unpinned transitives.** Reproducible until the
  day it is not.
- **`pip freeze` as a lockfile.** No hashes, single-platform, includes packages you did not
  ask for and omits markers.
- **`--extra-index-url` with a private and a public index.** Dependency confusion by
  configuration. Use a single proxying index.
- **Latest tags in container images.** Not reproducible by construction.
- **Pinning in a library.** Makes it uninstallable alongside anything else.
- **Never upgrading.** A frozen dependency tree accumulates unpatched CVEs and eventually
  a forced, enormous upgrade. Automate small, frequent upgrades (Renovate/Dependabot) with
  a good test suite as the gate.
- **Upgrading everything immediately.** No cooling-off window; you are the canary for
  compromised releases.
- **Adding a dependency for a function you could write.** §2.5 (7).
- **Not testing the installed artifact.** Tests that run against the source tree do not
  catch a missing `package_data` entry or a wrong `packages` config. Test against an
  installed wheel in CI.

## 5. Exercises

### Warm-up (25 min)

**W1.** Create a diamond conflict deliberately and read the resolver's error output.
Explain what it tells you and what it does not.

**W2.** Compare `pip freeze` output with a `uv.lock` for the same project. List every kind
of information the lock has that the freeze does not.

**W3.** Build a wheel and an sdist for a trivial package. Inspect both (`unzip -l`,
`tar tf`). Explain what each contains and why the wheel installs faster.

### Core (2 h)

**C1 — Reproducibility ladder.** Complete §3 for a real project. Deliverable: each rung
implemented or explicitly declined with a reason; the two-build digest comparison; a
`REPRODUCIBILITY.md` documenting exactly what is and is not guaranteed. That document is
the assessed artifact.

**C2 — Publish a package.** Take the Term 1 build artifact and publish it properly:
TestPyPI rehearsal, then PyPI, with trusted publishing from CI, `py.typed`, a CHANGELOG,
and a documented public-API boundary. Deliverable: the URL and the release workflow.

**C3 — Dependency audit.** For a real project, produce a table of every direct dependency
with: what it does, how much of its API you use, its maintainer count, its last release
date, its transitive dependency count, and whether you could remove it. Recommend at least
one removal with the diff.

**C4 — Supply-chain incident tabletop.** Write a runbook for "a package in our lockfile
published a malicious version". Cover: how you would find out, how you determine whether
you installed it, what you do about builds already deployed, and what changes afterward.
Then *test* the detection half by checking whether your current tooling would actually tell
you.

### Challenge

**X1.** Implement a small dependency resolver for a simplified model (packages, versions,
`>=`/`<` constraints). Use backtracking with conflict-driven clause learning if you are
ambitious. Then construct an input where naive backtracking takes exponential time and
explain the connection to SAT. (Return to this after CS-621 L09.)

**X2.** Take one of your Docker images and pursue bit-for-bit reproducibility as far as you
can in one working day. Document every source of non-determinism you found and how you
addressed or failed to address it. Compare your findings with the Reproducible Builds
project's documentation.

## 6. Self-check

1. What do PEPs 517, 518, and 621 each define?
2. Why should applications pin and libraries not?
3. Why is dependency resolution NP-complete, and what does Python's flat environment imply?
4. Name five things a lockfile does not guarantee.
5. Explain dependency confusion and the configuration that causes it.
6. Why do wheels reduce supply-chain risk compared to sdists?
7. Give the reproducibility ladder and say where a typical team should stop.
8. Why must you test against an installed wheel and not only the source tree?

## 7. Primary sources

- PEPs 517, 518, 621, 440, 508, 735, 561. The Python Packaging User Guide is the readable
  synthesis.
- `uv` documentation, particularly the resolution and locking chapters.
- Reproducible Builds project documentation (reproducible-builds.org).
- Winters et al., *Software Engineering at Google*, ch. 21 ("Dependency Management") — the
  clearest statement of why this is hard at scale.
- OpenSSF's "Concise Guide for Evaluating Open Source Software".

---

**Previous:** [L08](L08-static-analysis.md) · **Next:**
[L10 — CI as a Design Constraint](L10-ci-as-a-design-constraint.md)
