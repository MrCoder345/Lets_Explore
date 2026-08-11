# Lesson 16: Dictionaries & Hash Tables

Dictionaries are not just a data structure in Python; they *are* Python.

Every module's namespace is a dictionary. Every class definition is a dictionary. Every object's attributes are backed by a dictionary. Because the Virtual Machine relies on dictionaries to resolve almost every variable name and method call, `dictobject.c` is arguably the most heavily optimized file in the entire CPython codebase.

---

## 1. Lesson Overview

In this lesson, we dissect the modern CPython Dictionary. We explore:

- The **"Compact Dict"** architecture (introduced in Python 3.6) and how it guarantees insertion order.
- How hash collisions are resolved using **Open Addressing**.
- How CPython splits keys and values in memory to optimise object attributes (**Split‑Table Optimization – PEP 412**).

**Why it matters:**  
Because dictionaries map strings to objects, **O(1)** lookup speed is the foundation of Python's performance. As a C extension writer, you will interact with dictionaries constantly. If you misunderstand dictionary reference ownership, your code will inevitably crash.

**Prerequisites:** You understand `PyObject` headers (Lesson 8), Reference Counting (Lesson 9), and String Hashing (Lesson 14).

---

## 2. Mental Model – The Split Design

Before Python 3.6, a dictionary was a single, sparse C array. It wasted huge amounts of memory on empty slots and did not preserve insertion order.

Modern CPython splits the dictionary into **two** arrays:

1. **Indices (The Hash Table):** A small, sparse array of integers (8‑bit, 16‑bit, or 32‑bit). This array represents the hash table. Its slots are either empty (`-1`) or contain an index pointing into the Entries array.

2. **Entries (The Payload):** A dense, perfectly contiguous array of `PyDictKeyEntry` structs. Every time you add an item, it is appended exactly at the end of this array. This is why modern dicts preserve insertion order – they are physically appended in order!

```
d = {"a": 10, "b": 20}

[ Indices Array ] (Sparse, 8 slots)
 Slot 0: -1 (Empty)
 Slot 1:  1 ────┐
 Slot 2: -1     │
 Slot 3: -1     │
 Slot 4:  0 ─┐  │
 ...         │  │
             │  │   [ Entries Array ] (Dense, Contiguous)
             │  │   Index 0: { hash("a"), &PyUnicode("a"), &PyLong(10) } ◄──┐
             │  └──►Index 1: { hash("b"), &PyUnicode("b"), &PyLong(20) } ◄──┘
             └──────Index 2: (Empty, ready for next append)
```

---

## 3. Where We Are in the Repository

| **Path / File**                     | **Role**                                                                  |
|-------------------------------------|---------------------------------------------------------------------------|
| `Objects/dictobject.c`              | Implementation of dict operations – lookup, insertion, resizing.          |
| `Include/cpython/dictobject.h`      | Public API for dictionaries (`PyDict_GetItem`, `PyDict_SetItem`).          |
| `Include/internal/pycore_dict.h`    | Internal definitions – `PyDictKeysObject`, `PyDictKeyEntry`, etc.          |

**Why they matter:**  
- `dictobject.c` contains the probing and resizing logic.  
- `pycore_dict.h` contains the highly specialised structs that separate the keys from the values.

---

## 4. Concepts We Need First

### 4.1 Open Addressing with Perturbation
When two keys hash to the same slot, CPython does **not** use a linked list (Separate Chaining). Linked lists ruin CPU cache locality. Instead, CPython uses **Open Addressing**: it mathematically jumps to another slot in the Indices array. To prevent clustering, it uses a formula involving the higher bits of the hash:

```
j = ((j * 5) + perturb + 1)
```

This spreads out keys nicely across the table.

### 4.2 Split‑Table Optimization (PEP 412)
If you create 1,000 instances of `class Point`, all 1,000 instances have the exact same attribute names (`x` and `y`). It is a massive waste of memory to duplicate the keys `"x"` and `"y"` 1,000 times. CPython allows the 1,000 dictionaries to **share** a single `PyDictKeysObject` (the names), while each instance holds its own `PyDictValues` array (the actual data). This is called a **split‑table**.

---

## 5. Architecture

1. **The Header** – `PyDictObject` is just a shell. It holds:
   - `ob_refcnt`, `ob_type`.
   - `ma_used` – number of active items.
   - `ma_keys` – pointer to the underlying `PyDictKeysObject`.
   - `ma_values` – `NULL` for normal dicts; points to a separate values array for split dicts.

2. **Keys & Indices** – `ma_keys` points to a `PyDictKeysObject`. This struct holds:
   - The sparse **Indices** array.
   - The dense **Entries** array.

3. **Values** – For a standard dict, values are stored directly inside the Entries array. For a split‑table (an object's `__dict__`), `ma_values` points to a separate C array of `PyObject *` pointers.

4. **Lookup** – The lookup process:
   - Hash the key.
   - Modulo by table size to get initial index.
   - Read the Indices array to get an entry offset.
   - Read the Entries array at that offset.
   - Compare hash → compare pointer identity (`==`) → compare object equality (`richcompare`).

---

## 6. Important Data Structures

### `PyDictObject`
```c
typedef struct {
    PyObject_HEAD
    Py_ssize_t ma_used;          // Number of active items
    uint64_t ma_version_tag;     // Used for inline caching optimizations
    PyDictKeysObject *ma_keys;   // Pointer to the keys/hash structure
    PyObject **ma_values;        // NULL for normal dicts; array for split dicts
} PyDictObject;
```

### `PyDictKeyEntry` (the dense payload)
```c
typedef struct {
    Py_hash_t me_hash;  // Cached hash to avoid recomputing on resize
    PyObject *me_key;   // Strong reference to the key
    PyObject *me_value; // Strong reference to the value
} PyDictKeyEntry;
```

### `PyDictKeysObject` (the keys + indices container)
```c
struct _dictkeysobject {
    Py_ssize_t dk_refcnt;        // Reference count (shared across split dicts)
    Py_ssize_t dk_size;          // Size of the Indices array (power of 2)
    Py_ssize_t dk_nentries;      // Number of entries in the dense array
    uint8_t dk_indices[];        // Flexible array member – the sparse indices array
    // The Entries array follows immediately after dk_indices in memory
};
```

---

## 7. Important Functions

| Function                          | Purpose                                                                                                        |
|-----------------------------------|----------------------------------------------------------------------------------------------------------------|
| `PyDict_GetItem(p, key)`          | Retrieves a value. **CRITICAL:** Returns a **Borrowed Reference**. Prefer `PyDict_GetItemRef` in modern code. |
| `PyDict_SetItem(p, key, val)`     | Adds an item. Does **not** steal your references – it calls `Py_INCREF` on both key and value.                 |
| `lookdict_unicode()`              | Highly specialised internal function. Because 95% of dictionary lookups use string keys, CPython skips the generic `lookdict()` and uses this string‑optimised probing loop. |
| `dictresize(PyDictObject *mp, Py_ssize_t minused)` | Grows the hash table to a larger size and re‑inserts all items into the new Indices array. |

---

## 8. Important Macros

| Macro                                    | Purpose                                                                                         |
|------------------------------------------|-------------------------------------------------------------------------------------------------|
| `DK_ENTRIES(dk)`                         | Calculates the memory address of the Entries array based on the dynamic size of the Indices array (which can be 8‑bit, 16‑bit, or 32‑bit). |
| `PyDict_Check(op)`                       | Returns `1` if `op` is a dictionary.                                                            |

---

## 9. Source Code Exploration

1. **`Include/internal/pycore_dict.h`** – Search for `struct _dictkeysobject`. Look at the `dk_indices` char array at the end – it uses the C struct hack (flexible array member) we saw in tuples.

2. **`Objects/dictobject.c`** – Search for `lookdict_unicode`. Trace the `while` loop. You will see:
   - Read the index from `dk_indices`.
   - If empty, return `NULL`.
   - If the hash matches, check if the string pointers are exactly identical (fast path).
   - If pointers differ, call `unicode_eq` to compare characters (slow path).
   - If not a match, apply the `perturb` formula and repeat.

3. **`Objects/dictobject.c`** – Search for `dictresize`. See how it allocates a new, larger `PyDictKeysObject` and re‑inserts all items from the dense array into the new sparse hash table.

---

## 10. Execution Flow – Inserting a Key

Trace: `d = {}; d["x"] = 42`

1. **`d = {}`** – `PyDict_New()` is called. It allocates a `PyDictObject`. The `ma_keys` points to a static, read‑only empty keys object to save memory until the first insertion.

2. **`d["x"] = 42`** – `PyDict_SetItem` is called.

3. The dict realises it is using the dummy empty keys, so it allocates a new `PyDictKeysObject` with a capacity of 8.

4. It hashes the string `"x"` (or uses the cached hash from the `PyCompactUnicodeObject`).

5. It masks the hash (e.g., `hash % 8`) to find a slot in the Indices array.

6. The slot is empty (`-1`). It updates the slot to `0` (the first entry index).

7. It writes `{hash("x"), &"x", &42}` into index 0 of the Entries array.

8. It calls `Py_INCREF("x")` and `Py_INCREF(42)`.

9. It increments `ma_used`.

---

## 11. Real Python Example – Iteration Order

You can observe the dense array behaviour through iteration:

```python
d = {}
d["c"] = 1
d["a"] = 2
d["b"] = 3
del d["a"]
d["d"] = 4

print(list(d.keys()))   # ['c', 'b', 'd']
```

When you iterate over a dictionary, CPython literally just writes a `for` loop over the dense Entries array (`for i in range(dk_nentries)`). It ignores the sparse hash table entirely. If an entry's value is `NULL` (like `"a"` after deletion), it just skips it. This makes dictionary iteration **incredibly fast** compared to old CPython versions, which had to scan the sparse table.

---

## 12. Why This Design?

### Why store the hashes inside the Entries array?
When a dictionary grows (e.g., hits 2/3rds capacity), it must be resized. Resizing requires re‑mapping every single key into a new, larger Indices array. If CPython didn't store the hashes, it would have to recompute the hash for every object during a resize, which is computationally ruinous. Storing the hash makes resizing **O(N)** rather than **O(N × hash_cost)**.

### Why return a Borrowed Reference from `PyDict_GetItem`?
Performance. 99% of the time, the VM just wants to look up a function and call it immediately. Forcing the C code to `Py_INCREF` on get, and then `Py_DECREF` right after the call, adds massive overhead. *(However, in free‑threaded Python, borrowed references are fatal if another thread deletes the key, hence the shift to `PyDict_GetItemRef`, which returns a New Reference.)*

---

## 13. Common Beginner Mistakes

- **Mistake:** Storing the pointer from `PyDict_GetItem` without `INCREF`ing it.
  ```c
  PyObject *val = PyDict_GetItem(dict, key);
  // ... later ...
  PyDict_DelItem(dict, key);   // The dictionary releases 'val'
  printf("%s", PyUnicode_AsUTF8(val));   // SEGFAULT! Use‑After‑Free
  ```
  **Correction:** `val` is borrowed. If you need it to survive, `Py_INCREF(val)` immediately.

- **Mistake:** Mutating a dictionary while iterating over it in C.
  **Correction:** If you add an item during iteration, `dictresize` might trigger. The Entries array will be reallocated to a new memory address, and your C loop's pointers will now be pointing to freed memory.

- **Mistake:** Assuming `PyDict_SetItem` steals references.
  **Correction:** It increments the refcounts of both key and value. You still own your original references and must `Py_DECREF` them when done.

---

## 14. Summary

Modern CPython dictionaries use a highly memory‑efficient “compact” architecture:

- Split into a sparse **Indices** array (for **O(1)** hash probing) and a dense **Entries** array (which preserves insertion order and speeds up iteration).
- Dictionaries own strong references to their keys and values.
- The C API heavily relies on **borrowed references** for fast lookups (but modern free‑threaded builds are moving to new‑reference APIs).

---

## 15. Mental Model to Remember

```
(Lookup "x") -> Hash("x") -> Modulo Table Size
                     │
                     ▼
           [ Sparse Indices Array ]
          (Points to an offset, e.g., 2)
                     │
                     ▼
           [ Dense Entries Array ]
          Offset 0: { ... }
          Offset 1: { ... }
          Offset 2: { hash("x"), "x", 42 } ◄── Found!
```

---

## 16. Important Functions (Quick Reference)

| Function                     | Purpose                                                             |
|------------------------------|---------------------------------------------------------------------|
| `PyDict_GetItem()`           | Lookup – returns a **Borrowed Reference**. Use with caution.        |
| `PyDict_GetItemRef()`        | Modern alternative – returns a **New Reference** (safer).           |
| `PyDict_SetItem()`           | Insert/update – takes strong refs (does not steal).                |
| `lookdict_unicode()`         | Fast, specialised lookup for string keys.                          |
| `dictresize()`               | Internal resize routine – grows the Indices array.                 |

---

## 17. Important Structs

| Struct              | Purpose                                                           |
|---------------------|-------------------------------------------------------------------|
| `PyDictObject`      | The dictionary object – holds header, size, and pointers to data. |
| `PyDictKeysObject`  | Contains the sparse Indices array and the dense Entries array.    |
| `PyDictKeyEntry`    | One entry in the dense array – hash, key pointer, value pointer.  |

---

## 18. Important Files

| File                              | Role                                                         |
|-----------------------------------|--------------------------------------------------------------|
| `Objects/dictobject.c`            | Full implementation – lookup, insertion, deletion, resizing. |
| `Include/cpython/dictobject.h`    | Public C API.                                                |
| `Include/internal/pycore_dict.h`  | Internal definitions – `PyDictKeysObject`, `PyDictKeyEntry`. |

---

## 19. Code‑Reading Exercises

1. Open `Include/internal/pycore_dict.h`. Locate `PyDictKeysObject`. Notice the `dk_usable` and `dk_nentries` fields. `dk_usable` tracks how many more items can be inserted before a resize is triggered.

2. Open `Objects/dictobject.c`. Search for `PyDict_SetItem`. Follow its logic to see how it calls `Py_INCREF` on both the key and the value *before* inserting them.

3. In the same file, search for `insertdict`. This is the internal function that actually writes the `PyDictKeyEntry` into the dense array and updates the sparse index.

---

## 20. Understanding Questions

1. If you delete an item from a dictionary, CPython cannot simply shift all subsequent items in the dense Entries array to the left. **Why?**  
   *(Hint: Think about what the sparse Indices array points to.)*

2. If an object relies on the **Split‑Table** optimization (sharing `ma_keys` with 1,000 other objects), **what must happen dynamically** if you add a completely new attribute to *just one* of those objects?

3. Why does CPython check **pointer identity** (`if (key == entry->me_key)`) before calling the rich comparison function during a hash collision probe?

---

## 21. Suggested Next Files to Read

- **`Objects/setobject.c`** – Sets are mathematically very similar to dictionaries, but they don't have values. Let's see how they differ in memory.

---

## 22. Suggested Next Lesson

**Lesson 17 – Sets & Hashing**  
We will briefly cover sets, solidify our understanding of the `__hash__` protocol, and then move into Phase 5: how functions are actually called and executed.
