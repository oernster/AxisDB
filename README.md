# AxisDB

AxisDB is a tiny embedded document database for Python.

Key properties:

- Library-first design (usable without any server)
- Single-file JSON storage
- Atomic, crash-safe commits via temp-file replace
- Safe multi-process access (single writer, multiple readers)
- Minimal indexing and query support (MVP)

> **See also:** USE_CASES.md — a concise overview of practical applications and patterns enabled by this multidimensional JSON storage model.  
You can initialize the database with any number of dimensions and read/write values using coordinates.

---

## Features

- JSON file-based embedded document database
- N-dimensional coordinate keys
- Atomic commit and recovery
- Prefix key listing and basic querying
- Optional FastAPI wrapper with Swagger (`/docs`) and ReDoc (`/redoc`)

## Compatibility promises

This project follows semantic versioning for the public Python API.

Stable (backwards compatible within a major version):

- Public API surface of [`AxisDB`](axisdb/api.py:1): `open`, `create`, `get`, `set`, `delete`, `exists`, `list`, `slice`, `find`, `commit`, `rollback`.
- Public exception types in [`axisdb/errors.py`](axisdb/errors.py:1).
- On-disk file format `format=axisdb` and `format_version=2` in [`axisdb/engine/storage.py`](axisdb/engine/storage.py:1).

May change in minor versions:

- Internal module organization and private/internal attributes.
- Index query planning details (which predicates are index-accelerated).

Breaking changes will only occur in the next major version.

## Query and indexing (MVP, but useful)

AxisDB is optimized for correctness and simple, real-world scans.
Indexes are rebuilt on commit (durable, deterministic) and used to reduce scan work.

### Indexes

- Prefix index: always maintained on commit. Accelerates prefix scans.
- Field indexes: optional; define and persist on commit.

Define a field index:

```python
from axisdb import AxisDB

db = AxisDB.create("./mydb.json", dimensions=2)
db.define_field_index("by_customer_id", ("customer_id",))
```

### Killer query 1: prefix constraint + field equality

Example: all documents under the `("orders", *)` prefix for a specific customer.

```python
from axisdb.query.ast import Field

rows = db.find(prefix=("orders",), where=Field(("customer_id",), "==", "c2"))
```

Candidate selection behavior:

- If a matching field index exists for `("customer_id",)` and there is no uncommitted overlay, `find()` uses the field index then applies the prefix filter.
- Otherwise it falls back to prefix scanning (or full scan).

Complexity (high level):

- With field index: O(m) candidates where m is number of matching field values (plus prefix filter cost).
- Without field index: O(k) where k is number of keys in the prefix range.

### Killer query 2: prefix-only scan + richer predicate

Example: scan one prefix range and apply a richer predicate.

```python
rows = db.find(
    prefix=("orders",),
    where=lambda doc: doc.get("amount", 0) >= 100,
)
```

Limits:

- Callable predicates are always a scan of the candidate set; they are not index-accelerated.
- Field index acceleration currently applies only to simple equality: `Field(path) == literal`.

---

## Installation

## 1. Create a virtual environment

### Windows (PowerShell)

```powershell
python -m venv venv
venv\Scripts\activate
```

### Linux / macOS / WSL

```bash
python3 -m venv venv
source venv/bin/activate
```

---

### 2. Install dependencies

From within the activated venv:

This project uses `uv` for environment management.

```powershell
uv venv .venv
uv pip install -r requirements.txt --python .venv\Scripts\python.exe
```

## Library usage

Create/open a database and commit writes:

```python
from axisdb import AxisDB

db = AxisDB.create("./mydb.json", dimensions=2)
db.set(("user1", "orders"), {"count": 3})
db.commit()

ro = AxisDB.open("./mydb.json", mode="r")
print(ro.get(("user1", "orders")))
```

## Running the FastAPI wrapper

Inside the project folder:

```powershell
.\.venv\Scripts\python.exe -m uvicorn axisdb.server.app:app --reload
```

The server will start at:

```
http://localhost:8000
```

---

## API Documentation

FastAPI automatically exposes interactive documentation:

### Swagger UI (interactive)
```
http://localhost:8000/docs
```

### ReDoc (static reference docs)
```
http://localhost:8000/redoc
```

---

## Typical API Usage

### Initialize a DB

The FastAPI wrapper uses query parameters for simplicity.

```bash
curl -X POST "http://localhost:8000/init?path=./mydb.json&dimensions=3&overwrite=true"
```

---

### Set a value

```bash
curl -X POST "http://localhost:8000/item?path=./mydb.json" \
  -H "Content-Type: application/json" \
  -d '{"coords": ["user1", "2025-01-01", "orders"], "value": {"order_id": 123, "amount": 49.99}}'
```

---

### Get an item

```bash
curl "http://localhost:8000/item?path=./mydb.json&coords=user1&coords=2025-01-01&coords=orders"
```

---

### List by prefix

```bash
curl "http://localhost:8000/list?path=./mydb.json&prefix=user1"
```

---

### Delete an item

```bash
curl -X DELETE "http://localhost:8000/item?path=./mydb.json" \
  -H "Content-Type: application/json" \
  -d '{"coords": ["user1", "2025-01-01", "orders"]}'
```

