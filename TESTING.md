# Testing

How AxisDB is tested: running the checks, reading what they say, what the suite
covers and what it leaves out; how a new test is written. How the library
is built is in [ARCHITECTURE.md](ARCHITECTURE.md); setting up the environment
the tests run in is in [DEVELOPMENT.md](DEVELOPMENT.md).

## Running the checks

From the repository root, with the venv active:

```powershell
python -m pytest -q
python -m black --check .
python -m ruff check .
```

There is no CI, so nothing runs these for you; run all three and read the exit
code of each. ruff is configured in `pyproject.toml` with a wider rule set than
its defaults (pycodestyle, pyflakes, import order, pyupgrade, bugbear and
simplify).

**A full run takes a few seconds.** Measured on 2026-10-02: 25 tests passed in
3 seconds on Windows.

**Read the exit code.** `0` means every test passed. There is no coverage gate
here, so the summary line is the result; `$LASTEXITCODE` is still the thing to
trust in a script.

## What the suite holds and leaves out

Testing emphasises real file IO and real multiprocess behaviour rather than
stand-ins: every test that touches storage writes a real database file into
pytest's `tmp_path`; the locking test runs real separate processes against
one file.

- **No coverage measurement.** pytest-cov is neither installed nor configured,
  so there is no floor and no figure.
- **The FastAPI wrapper is not tested.** Nothing under `tests/` exercises
  `axisdb/server`; the wrapper is a thin translation onto the library, whose
  behaviour the suite does cover. The translation itself is checked by
  nothing.

## Where the tests live

All six files sit flat in `tests/`:

| File | What it holds |
|---|---|
| `test_storage_recovery.py` | recovery on open: which of the main file and a commit's temporary file is kept when they disagree; a refusal when neither is valid |
| `test_locking_multiprocess.py` | across real processes, two writers cannot open together while a reader can open beside a writer |
| `test_api_basic.py` | the public operations: create, set, get, commit, rollback, read-only refusal, input validation, `list` and `find` |
| `test_find_indexed.py` | `find` using a field index for a simple equality, falling back when none matches |
| `test_slice.py` | `slice` with exact matches, wildcards and membership selectors |
| `test_keycodec.py` | the encoding of N-dimensional keys, pure |

## Writing a test

- **Real files, not fakes.** Open the database on a path under `tmp_path`;
  durability and locking are what this library is for. A stand-in for the
  file would test the stand-in.
- **No mocking library.** None is used anywhere in the suite.
- **More than one process.** Follow `test_locking_multiprocess.py`: start real
  processes against the same file rather than simulating contention in one.

---

See also [README.md](README.md), [ARCHITECTURE.md](ARCHITECTURE.md) and
[DEVELOPMENT.md](DEVELOPMENT.md).
