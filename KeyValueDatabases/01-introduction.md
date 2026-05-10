# Key-Value Databases — Introduction

A **Key-Value Database** (also called a Key-Value Store) is the simplest form of NoSQL database. It stores data as a collection of **key-value pairs**, where:

- The **key** is a unique identifier (a string, integer, UUID, etc.).
- The **value** is any arbitrary data associated with that key — a string, number, JSON document, binary blob, or even a complex data structure.

Think of it like a dictionary (or hash map) at massive scale, optimised for extremely fast reads and writes.

```
┌─────────────────────────────────────────────────┐
│              Key-Value Store                    │
│                                                 │
│   "user:1001"    →   {"name": "Rahul",          │
│                        "age": 30}               │
│                                                 │
│   "session:abc"  →   "eyJhbGciOiJIUzI1NiJ9..." │
│                                                 │
│   "counter:hits" →   42819                      │
│                                                 │
│   "product:555"  →   {"title": "Laptop",        │
│                        "price": 999.99}         │
└─────────────────────────────────────────────────┘
```

## Simple Code Example

Below is a conceptual Python example showing how data is stored and retrieved using key-value pairs. This mimics what a real KV store does internally.

```python
# Conceptual representation of a Key-Value Store as a Python dict
store = {}

# --- WRITE ---
store["user:1001"] = {"name": "Rahul", "age": 30, "city": "Bangalore"}
store["session:abc123"] = "eyJhbGciOiJIUzI1NiJ9.payload.signature"
store["counter:page_hits"] = 42819

# --- READ ---
user = store.get("user:1001")
print(user)  # {'name': 'Rahul', 'age': 30, 'city': 'Bangalore'}

session = store.get("session:abc123")
print(session)  # eyJhbGciOiJIUzI1NiJ9.payload.signature

# --- DELETE ---
del store["session:abc123"]

# --- CHECK EXISTENCE ---
if "user:1001" in store:
    print("User found!")
```

Key observations:
- The lookup is **O(1)** — no scanning, no joins.
- The key is the *only* way to access a value (no queries by field).
- Values are **opaque** to the store — it does not parse or index their contents.

---

**Navigation:**
- [← Back to Index](README.md)
- [Next: Design Principles →](02-design-principles.md)
