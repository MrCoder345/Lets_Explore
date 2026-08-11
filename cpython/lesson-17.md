# Lesson 17: Sets & Hashing

In Lesson 16, we saw how modern dictionaries use a split architecture to save memory and preserve insertion order.

You might assume that `set` objects use the exact same architecture, just without the values. **They do not.** Sets retain the older, pre‑3.6 classic hash table design. Understanding this difference is crucial for mastering CPython's overall object landscape.

---

## 1. Lesson Overview

In this lesson, we study the internal C representation of Python sets (`set` and `frozenset`). We learn:

- How a `PySetObject` implements a classic **sparse hash table**.
- How it resolves collisions using **Open Addressing**.
- How it handles deletion using **“dummy” nodes** (tombstones) to preserve the probing chain.

**Why it matters:**  
If you are deduplicating massive datasets, sets are your primary tool. Understanding why sets do *not* preserve insertion order (unlike dictionaries) and how they handle deletions will prevent subtle bugs and performance traps in your code.

**Prerequisites:** You understand Dictionary internals and Hash probing (Lesson 16).

---

## 2. Mental Model – The Classic Sparse Table

A `set` is a **single, continuous C array** of `setentry` structs. Unlike modern dictionaries, there is **no dense array** tracking the order of insertion. The keys are dropped directly into the sparse hash table itself.

Because the table is sparse (partially empty), when you iterate over a set, CPython just walks sequentially through memory, skipping the empty slots. This is why **set iteration order is arbitrary** and changes if the set is resized!

```
s = {"a", "b"}

[ Table Array (8 slots) ]
 Index 0: { hash("a"), &PyUnicode("a") }
 Index 1: (Empty)
 Index 2: (Empty)
 Index 3: { hash("b"), &PyUnicode("b") }
 Index 4: (Empty)
 ...
```

---

## 3. Where We Are in the Repository

| **Path / File**                     | **Role**                                                                  |
|-------------------------------------|---------------------------------------------------------------------------|
| `Include/cpython/setobject.h`       | Definition of `PySetObject` and `setentry`.                               |
| `Objects/setobject.c`               | Implementation of set operations – add, remove, union, intersection, etc. |

**Why they matter:**  
- `setobject.h` defines the C structs.  
- `setobject.c` contains the probing logic and the mathematical set operations.

---

## 4. Concepts We Need First

### 4.1 Hash Collisions and Open Addressing
Like dicts, if two keys hash to the same index, CPython calculates a new index using a mathematical perturbation formula and jumps there. This forms a **“probe chain”**.

### 4.2 Dummy Nodes (Tombstones)
If `A` and `B` collide, `B` is placed further down the probe chain. If you later delete `A`, you cannot just leave the slot `Empty`. If you did, a search for `B` would hit the `Empty` slot, assume `B` doesn't exist, and fail. Instead, you replace `A` with a special **Dummy** object. The search algorithm knows to skip over Dummies and keep probing.

---

## 5. Architecture

1. **The Header** – `PySetObject` tracks:
   - `used` – number of active items.
   - `fill` – number of active items + dummies.
   - `mask` – table size minus 1.
   - `table` – pointer to the setentry array.

2. **Small Set Optimisation** – To avoid calling `malloc` for small sets, the `PySetObject` struct contains a pre‑allocated array of **8 `setentry` slots** directly inside the struct (`smalltable[8]`).

3. **Heap Array** – If the set grows beyond 8 items (technically, beyond a 3/5ths load factor), it allocates a larger array on the heap and updates its `table` pointer.

4. **Lookup (`set_lookkey`)** – Hash the key → modulo by table size → check slot → if match, return. If Dummy or mismatch, perturb hash and probe again.

---

## 6. Important Data Structures

### `setentry` – One Slot in the Hash Table
```c
typedef struct {
    PyObject *key;
    Py_hash_t hash;
} setentry;
```

### `PySetObject`
```c
typedef struct {
    PyObject_HEAD
    Py_ssize_t fill;            // Number of active + dummy entries
    Py_ssize_t used;            // Number of active entries
    Py_ssize_t mask;            // Table size minus 1 (for modulo)
    setentry *table;            // Pointer to array (internal or heap)
    Py_hash_t hash;             // Only used by frozenset
    Py_ssize_t finger;          // Used to optimise set.pop()
    setentry smalltable[8];     // Pre-allocated space for small sets
    PyObject *weakreflist;      
} PySetObject;
```

---

## 7. Important Functions

| Function                                    | Purpose                                                                                               |
|---------------------------------------------|-------------------------------------------------------------------------------------------------------|
| `set_lookkey(PySetObject *so, PyObject *key, Py_hash_t hash)` | The core of `setobject.c`. Returns a pointer to the `setentry`. If the key doesn't exist, returns a pointer to the empty slot where the key *should* go. |
| `set_add_entry(PySetObject *so, PyObject *key, Py_hash_t hash)` | Calls `set_lookkey`. If the slot is empty or dummy, inserts the key and increments `used`. Checks load factor and calls `set_table_resize` if needed. |
| `set_discard_entry(...)`                    | Calls `set_lookkey`. If found, drops the reference to the key (`Py_DECREF`) and writes the global `<dummy>` object pointer into the `key` field. |

---

## 8. Important Macros

| Macro                   | Purpose                                               |
|-------------------------|-------------------------------------------------------|
| `PySet_GET_SIZE(so)`    | Returns the `used` field (number of active items).    |
| `dummy`                 | A static global `PyObject` used as a unique memory address marker (a tombstone). |

---

## 9. Source Code Exploration

1. **`Include/cpython/setobject.h`** – Look at `PySetObject`. Notice the `smalltable[8]` array physically embedded inside the struct. This is a massive optimisation for short‑lived, tiny sets (like `{1, 2, 3}`).

2. **`Objects/setobject.c`** – Search for `set_lookkey`. Read the `while` loop. You will see:
   ```c
   if (entry->key == NULL) return entry;              // Found empty slot
   if (entry->key == dummy) { ... }                   // Mark dummy for possible reuse, but keep probing
   if (entry->hash == hash && PyObject_RichCompareBool(entry->key, key, Py_EQ)) return entry; // Found it!
   // ... perturb and repeat
   ```

3. **`Objects/setobject.c`** – Search for `set_pop`. Look at how it uses `so->finger` to remember where it left off during the last `pop()`, so it doesn't have to scan the entire array from index 0 every time.

---

## 10. Execution Flow – Adding and Removing

Trace: `s = set(); s.add("a"); s.remove("a")`

1. **`set()`** – `PySet_New(NULL)` is called.  
   - `so->table` is set to point to `so->smalltable`.  
   - All 8 slots are zeroed out.

2. **`s.add("a")`**
   - Hash `"a"`.
   - `set_lookkey` finds an empty slot.
   - `Py_INCREF("a")`.
   - Slot updated: `key = "a"`, `hash = hash("a")`.
   - `used` increments to 1. `fill` increments to 1.

3. **`s.remove("a")`**
   - Hash `"a"`.
   - `set_lookkey` finds the slot.
   - The C code copies the pointer to `"a"`.
   - Slot updated: `key = dummy`. (The hash remains – it doesn't matter.)
   - `used` decrements to 0. `fill` remains 1 (a dummy still takes up capacity!).
   - `Py_DECREF("a")` is called.

---

## 11. Real Python Example – Ordering

Compare dict and set ordering:

```python
# Dicts preserve insertion order (dense array)
d = {}
d[1] = None
d[100] = None
print(list(d.keys()))   # [1, 100]

# Sets do NOT preserve insertion order (sparse array only)
s = set()
s.add(1)
s.add(100)
print(list(s))          # Output might be [1, 100] or [100, 1], depending on hash collisions and table size!
```

Because the set drops items randomly into the 8‑slot array based on their hash modulo, iteration simply reads the array from index 0 to 7. The order you see is the physical memory layout.

---

## 12. Why This Design?

### Why didn't Sets get the “Compact” redesign that Dicts got in 3.6?
Dictionaries are used for `**kwargs` and class definitions, where preserving order is highly desirable for developers. Sets are mathematical constructs – order is inherently meaningless. Implementing the dense/sparse split for sets would increase memory complexity without providing a mathematically necessary feature.

### Why use `smalltable[8]` inside the struct?
Unlike dictionaries (which often grow large), sets are frequently used temporarily for quick membership testing (`if x in {1, 2, 3}:`). The small table entirely avoids `pymalloc` heap allocation for the payload, keeping the entire operation within a single CPU cache line.

---

## 13. Common Beginner Mistakes

- **Mistake:** Putting a list inside a set (`s = {[1, 2]}`).  
  **Correction:** You will get a `TypeError: unhashable type: 'list'`. Why? The set relies on `hash()`. If a list's contents change, its hash would change, meaning it would be in the wrong slot in the `setentry` array. You would never be able to find it again! Only immutable objects (where the hash is constant) can be placed in sets.

- **Mistake:** Assuming `frozenset` has a different internal C layout than `set`.  
  **Correction:** They use the exact same `PySetObject` struct. The only difference is that `frozenset` actually uses the `so->hash` field (because it can be hashed and put inside *another* set), and the C API prevents you from calling mutation functions on it.

---

## 14. Summary

Python sets are classic, sparse hash tables powered by Open Addressing. They:

- Use an embedded 8‑slot array for extreme performance on small datasets.
- Dynamically allocate larger heap arrays as they grow.
- Utilise **dummy nodes** (tombstones) to maintain probing chains after deletion.
- **Do not** preserve insertion order – iteration order is arbitrary and relies purely on the physical hash layout.

---

## 15. Mental Model to Remember

```
PySetObject
 ├── used: 2
 ├── table: ─┐
 └── smalltable [ (Empty), (Dummy), ("A"), (Empty), ("B"), ... ]
             ▲
             └─ Iteration just walks this array sequentially.
```

---

## 16. Important Functions (Quick Reference)

| Function            | Purpose                                            |
|---------------------|----------------------------------------------------|
| `set_lookkey()`     | Core probing – finds a key or an empty slot.       |
| `set_add_entry()`   | Inserts a key (calls `set_lookkey` + resize).      |
| `set_discard_entry()` | Removes a key (marks slot with `dummy`).         |

---

## 17. Important Structs

| Struct         | Purpose                                                 |
|----------------|---------------------------------------------------------|
| `setentry`     | One slot – hash + key pointer.                          |
| `PySetObject`  | The set object – header, table pointer, smalltable, etc. |

---

## 18. Important Files

| File                           | Role                                                   |
|--------------------------------|--------------------------------------------------------|
| `Include/cpython/setobject.h`  | Definition of `PySetObject` and `setentry`.            |
| `Objects/setobject.c`          | Implementation of lookup, insertion, deletion, and set operations. |

---

## 19. Code‑Reading Exercises

1. Open `Include/cpython/setobject.h`. Look at `PySetObject`. Compare it mentally to `PyDictObject`. Notice the lack of a separate “Keys” struct.

2. Open `Objects/setobject.c`. Search for the `dummy` object definition near the top. Note how it is literally just a statically allocated `PyObject` whose only purpose is to have a unique memory address.

3. In the same file, search for `set_pop`. Look at how it uses `so->finger` to scan through the table, resetting to `0` when it reaches the end, to efficiently find and pop an element without doing O(N) scans every time.

---

## 20. Understanding Questions

1. If a set's `smalltable` holds 8 items, **why does CPython trigger a resize/allocation when the set hits 5 items** instead of waiting until it hits 8?

2. If you add 1,000 items to a set, and then remove 1,000 items, the `used` count is 0. But **what is the `fill` count**, and why does this impact the speed of future lookups?

3. Why does `frozenset` **cache its hash value** in the `so->hash` field, but `set` does not?
