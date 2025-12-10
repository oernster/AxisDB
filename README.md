# Multidimensional JSON NoSQL REST API

A FastAPI-powered REST server that stores **N-dimensional JSON data** inside a single JSON file.  
You can initialize the database with any number of dimensions and read/write values using coordinates.

---

## 🚀 Features
- JSON file–based NoSQL-style storage  
- Arbitrary **N-dimensional** hierarchical keys  
- REST API for CRUD + slicing  
- Auto-generated **Swagger UI** (`/docs`)  
- Auto-generated **ReDoc** (`/redoc`)  

---

# 🔧 Installation

## 1. Create a virtual environment

### **Windows (PowerShell)**

```powershell
python -m venv venv
venv\Scripts\activate
```

### **Linux / macOS / WSL**

```bash
python3 -m venv venv
source venv/bin/activate
```

---

## 2. Install dependencies

From within the activated venv:

```bash
pip install -r requirements.txt
```

Your `requirements.txt` should contain:

```
fastapi
uvicorn
```

---

# ▶️ Running the Server

Inside the project folder:

```bash
uvicorn app:app --reload
```

Server will start at:

```
http://localhost:8000
```

---

# 📘 API Documentation

FastAPI automatically exposes interactive documentation:

### **Swagger UI (interactive)**
```
http://localhost:8000/docs
```

### **ReDoc (static reference docs)**
```
http://localhost:8000/redoc
```

---

# 🧪 Typical API Usage

### **Initialize DB with N dimensions**
```bash
curl -X POST http://localhost:8000/init   -H "Content-Type: application/json"   -d '{"dimensions": 3}'
```

---

### **Set a value at 3D coordinate**
```bash
curl -X POST http://localhost:8000/item   -H "Content-Type: application/json"   -d '{
        "coords": ["user1", "2025-01-01", "orders"],
        "value": {"order_id": 123, "amount": 49.99}
      }'
```

---

### **Get an item**
```bash
curl "http://localhost:8000/item?coords=user1&coords=2025-01-01&coords=orders"
```

---

### **Slice a prefix**
```bash
curl -X POST http://localhost:8000/slice   -H "Content-Type: application/json"   -d '{"prefix": ["user1"]}'
```

---

### **Delete an item**
```bash
curl -X DELETE http://localhost:8000/item   -H "Content-Type: application/json"   -d '{"coords": ["user1", "2025-01-01", "orders"]}'
```

---

# 📂 Project Structure (recommended)

```
multidb/
├─ app.py
├─ db.py
├─ schemas.py
├─ config.py
├─ requirements.txt
└─ README.md
```

