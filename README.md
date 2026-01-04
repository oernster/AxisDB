# AxisDB

**AxisDB** is a tiny embedded document database for Python, designed for simple, reliable storage of JSON documents addressed by **N-dimensional coordinate keys**.

It is library-first, requires no server, and stores all data in a single JSON file with atomic, crash-safe commits.

> **See also:**  
> [USE_CASES.md](https://github.com/oernster/AxisDB/blob/main/USE_CASES.md) — a concise overview of practical applications and patterns enabled by this multidimensional JSON storage model.

---

## Key properties

- Library-first design (usable without any server)
- Single-file JSON storage
- Atomic, crash-safe commits via temp-file replace
- Safe multi-process access (single writer, multiple readers)
- Minimal but useful indexing and query support (MVP)

---

## Features

- Embedded JSON document database
- N-dimensional coordinate keys
- Atomic commit and recovery
- Prefix key listing and basic querying
- Optional FastAPI wrapper with Swagger (`/docs`) and ReDoc (`/redoc`)

---

## Installation

### Core library

```bash
pip install axisdb
```

### Optional server components

```bash
pip install "axisdb[server]"
```

---

## Basic library usage

Create a database, write a value, and commit:

```python
from axisdb import AxisDB

db = AxisDB.create("./mydb.json", dimensions=2)
db.set(("user1", "orders"), {"count": 3})
db.commit()

ro = AxisDB.open("./mydb.json", mode="r")
print(ro.get(("user1", "orders")))
```

---

## Query and indexing (MVP)

AxisDB is optimized for correctness and predictable performance over complex query planning.  
Indexes are rebuilt on commit and used to reduce scan work when possible.

### Index types

- **Prefix index** — always maintained; accelerates prefix scans
- **Field indexes** — optional, user-defined, persisted on commit

Define a field index:

```python
from axisdb import AxisDB

db = AxisDB.create("./mydb.json", dimensions=2)
db.define_field_index("by_customer_id", ("customer_id",))
```

---

## Running the FastAPI wrapper (optional)

After installing the server extras:

```bash
pip install "axisdb[server]"
```

Run the server:

```bash
python -m uvicorn axisdb.server.app:app --reload
```

The server will start at:

```
http://localhost:8000
```

---

## API documentation

FastAPI automatically exposes interactive documentation:

- Swagger UI: `http://localhost:8000/docs`
- ReDoc: `http://localhost:8000/redoc`

---

## Versioning and compatibility

This project follows **semantic versioning**.

Guaranteed stable within a major version:

- Public `AxisDB` API: `open`, `create`, `get`, `set`, `delete`, `exists`, `list`, `slice`, `find`, `commit`, `rollback`
- Public exception types in `axisdb.errors`
- On-disk file format (`format=axisdb`, `format_version=2`)

May change in minor versions:

- Internal module structure and private attributes
- Index query planning details

Breaking changes will only occur in the next major version.
