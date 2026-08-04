# AxisDB: Technical Debt

A standing reference to the project's outstanding technical debt. It records what is still open, weighs whether each item is worth doing and gives the rationale. Every item is a behaviour-preserving internal concern: nothing here proposes changing the public API or the on-disk format. Scope is the whole repository: the `axisdb` package, the optional FastAPI server, the packaging metadata and the documentation.

The framing here is different from an application's. AxisDB is the only published library in the portfolio (`pip install axisdb`), so debt is weighed by what it costs *a consumer* rather than by what it costs to maintain the repository. A library's tests, its version and its README are part of its interface, not internal concerns.

---

## 1. A published library with no coverage gate

There is no `--cov-fail-under`, no `.coveragerc`, no `[tool.pytest.ini_options]` and no coverage configuration of any kind. `pytest` runs six test files against 1,655 lines and reports nothing about what it did not reach.

Every application in this portfolio carries a 100% gate, several of them scoped honestly to the surface that matters. The one artefact that other people install has none. That is the wrong way round: a defect in a desktop clock inconveniences its author, and a defect in a storage library silently corrupts someone else's data.

The library is small enough that a full gate is realistic rather than aspirational. Add:

```toml
[tool.pytest.ini_options]
addopts = "-q --cov=axisdb --cov-branch --cov-report=term-missing --cov-fail-under=100"
```

and omit `axisdb/server/*` if the FastAPI wrapper is not worth gating (it is an optional extra, so that is defensible). The engine, the key codec, the indexes and the locking are the product and should be at 100%.

The six existing tests are well chosen (`test_keycodec`, `test_slice`, `test_find_indexed`, `test_storage_recovery`, `test_locking_multiprocess`, `test_api_basic`) and cover the hard parts. This item is about knowing what they miss, not about doubting them.

## 2. The library does not expose its own version

`axisdb/__init__.py` exports `AxisDB` and nothing else. `pyproject.toml` holds `version = "1.0.6"` and it is the only version string in the repository.

A consumer cannot ask the library what version it is. `axisdb.__version__` does not exist, so a bug report against AxisDB cannot include the version without the reporter going to their package metadata, and any code branching on library version has to use `importlib.metadata`.

The minimal-public-API rule is right and `__all__` holding one name is right. `__version__` is not API bloat; it is the one attribute every published package is expected to carry. Add a `VERSION` file at root, read it in `axisdb/version.py` with a `0.0.0-dev` fallback, make `pyproject.toml` dynamic and re-export `__version__` alongside `AxisDB`.

The commit history shows the cost of the current arrangement directly: four of the last eight commits are "Bump version". That is a manual edit to `pyproject.toml` and nothing else, which is exactly the step a `VERSION` file removes.

## 3. Six em dashes, five of them in the PyPI long description

`README.md` is declared as `readme = "README.md"` in `pyproject.toml`, so it is rendered as the project page on PyPI. It contains five em dashes (lines 8, 83, 85, 158 and 159), and `ARCHITECTURE.md` contains a sixth in its own title.

The em-dash ban is absolute across the portfolio, and these are the most publicly visible instances of it anywhere in the account. Replace with a colon, a comma or parentheses. Fifteen minutes, and it is the front page of the one thing here that strangers install.

## 4. `axisdb/api.py` is 509 lines

The single largest module in the repository and the only one over 400. It is the public facade, so it legitimately carries every entry point (`list`, `slice`, `find`, index management, session lifecycle, commit), and a facade is expected to be wide.

Wide is not the same as long. 509 lines means the facade is also implementing rather than delegating, and it is the file a consumer reads to understand the library. Splitting the index-management and the query surfaces into sibling modules that `api.py` delegates to would take it under the cap and make the facade readable in one sitting.

There is also no structural test asserting the cap, so nothing reports this. A single size assertion beside the existing tests would cost almost nothing at this scale.

## 5. Nothing enforces the layering

`axisdb/engine/` (storage, key codec, indexes) and `axisdb/server/` (the FastAPI wrapper) are cleanly separated today, and `api.py` sits between them. Nothing holds that separation.

The specific risk is one direction: `axisdb/engine/*` must never import `axisdb/server/*` or `fastapi`, because `fastapi` is an optional extra. If that import ever appeared, the base install would break for every consumer who did not ask for the server, and only a fresh environment without FastAPI installed would reveal it. The developer's own environment has FastAPI, so the test suite would stay green.

One source-scan assertion (`axisdb/engine` and `axisdb/api.py` import nothing from `axisdb.server`, `fastapi` or `uvicorn`) closes a failure mode that is invisible locally and immediate for users. That makes it the highest value-per-line item in this file after item 1.

---

## Looks like debt, not worth touching

- The eight `except Exception as exc:  # noqa: BLE001` handlers in `axisdb/server/app.py`. They are the HTTP boundary of an optional wrapper, turning engine errors into responses. Broad is correct at that seam and each already carries the marker.
- `axisdb/engine/storage.py:109`'s broad handler. It is the crash-safe write path, where the whole point is to fail the commit cleanly rather than leave a half-written file. Already marked.
- `smoke_db.json`, its two lock files and `venv_smoke/` at repository root. Untracked smoke-test residue. Ignored correctly; only clutter in a working tree.
- The `[tool.ruff]` configuration selecting `E`, `F`, `W`, `I`, `UP`, `B`, `SIM` while the repository has no CI to run it. Good configuration waiting for item 1's infrastructure; adding a workflow that runs `ruff check` and `pytest` would serve both.
- `USE_CASES.md` alongside `README.md`. Deliberate, and the pattern this library established.

## Not debt (do not "fix" these)

These look like candidates but are correct as they stand; changing them would regress or add cost for nothing.

- **The single-name `__all__` in `axisdb/__init__.py`.** The minimal-public-API rule, correctly applied. Item 2 asks for `__version__` beside it and nothing more; the engine internals stay private.
- **`portalocker` as the only runtime dependency**, with `fastapi` and `uvicorn` behind a `server` extra. A storage library that pulls in a web framework by default would be a defect. This split is the library's most important packaging decision.
- **Single-writer, multi-reader file locking with `*.writer.lock` and `*.rw.lock`.** Two lock files looks like duplication; they have different semantics (exclusive for the writer session, shared during reads and exclusive during commit) and both are needed.
- **Atomic writes via a temporary file and `os.replace()`.** The crash-safety guarantee. Do not simplify it.
- **`license = "GPL-3.0-only"` with `license-files`, and the comment explaining why the TOML-table form was dropped.** Correct modern setuptools metadata, and the comment records why it changed.
- **GPL-3.0 rather than LGPL for a library.** Deliberate portfolio position: the licence is chosen by intent, not by artefact type.
