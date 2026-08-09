# Lesson 12: Object‑Specific Free Lists

You have seen how `pymalloc` provides raw C bytes extremely fast. But CPython is greedy for performance. Even O(1) byte allocation is sometimes too slow if you are creating and destroying the exact same types of objects millions of times per second.

To push performance to the absolute limit, CPython's built‑in types implement an *additional* layer of caching directly on top of `pymalloc` called **Free Lists**.

---

## 1. Lesson Overview

In this lesson, we study **Object‑Specific Free Lists**. We learn how built‑in types like `list`, `dict`, `float`, and `tuple` intercept their own destruction to recycle C structs, completely bypassing `pymalloc` and generic object initialisation.

**Why it matters:**  
If you are benchmarking Python code, you will see bizarre performance cliffs where creating 80 lists is incredibly fast, but creating 81 lists suddenly slows down. You must understand free lists to understand Python's actual runtime memory footprint and allocation speed.

**Prerequisites:** You understand `pymalloc` (Lesson 11) and `PyObject` lifecycles (Lesson 8).

---

## 2. Mental Model – The Recycling Bin

Think of `pymalloc` as a lumber yard providing raw wood (bytes). Think of generic object initialisation as a carpenter building a chair (a `PyListObject`).

If you throw away a chair, it is a waste of time to dismantle it into raw wood, return it to the lumber yard, and then ask the carpenter to build a *new* chair 10 microseconds later.  
Instead, when a list is destroyed, CPython simply puts the intact chair in a **“recycling bin”** (the Free List). When you need a new list, CPython grabs one from the bin, wipes the dust off (zeros the internal fields), and hands it back.

```
       del my_list                         new_list = []
           │                                     ▲
           ▼                                     │
   [ list_dealloc() ]                  [ PyList_New() ]
           │                                     │
   Is bin full? (>= 80)                Is bin empty? (== 0)
   ├── YES: Call pymalloc_free()       ├── YES: Call pymalloc_alloc()
   │                                   │
   └── NO: Push to bin                 └── NO: Pop from bin
           │                                     │
           ▼                                     ▲
     ┌─────────────────────────────────────────────┐
     │  List Free List (Array of PyListObject *)   │
     └─────────────────────────────────────────────┘
```

---

## 3. Where We Are in the Repository

| **Path / File**                         | **Role**                                                                 |
|-----------------------------------------|--------------------------------------------------------------------------|
| `Objects/listobject.c`                  | Free‑list logic for lists.                                               |
| `Objects/tupleobject.c`                 | Free‑list logic for tuples (multiple lists, one per length).             |
| `Include/internal/pycore_freelist.h`    | Internal definition of free‑list structures.                             |
| `Include/internal/pycore_interp.h`      | Where the free lists are stored (per‑interpreter, not global).           |

**Why they matter:**  
- The specific types (`listobject.c`) implement the push/pop logic.  
- In modern CPython, the actual arrays storing the free lists are centralised in the **Interpreter State** (`_PyInterpreterState`) to support multi‑threading and sub‑interpreters.

---

## 4. Concepts We Need First

### Multi‑Interpreter Isolation
In older versions of CPython, a free list was just a global C array:  
`static PyListObject *free_list[80];`

However:
- A global array is **not thread‑safe** without the GIL.
- It breaks isolation if you run multiple Python interpreters in the same process.

In modern CPython, free lists are stored inside a **`PyInterpreterState`** (or `_PyThreadState`) struct, so each interpreter has its own independent cache.

---

## 5. Architecture

1. **The Cache** – CPython maintains pre‑allocated arrays (or linked lists) of dead objects. For example:
   - The **List** free list holds up to **80** `PyListObject` pointers.
   - The **Tuple** free list is actually an array of **20** free lists, categorised by tuple length (a free list for 1‑item tuples, 2‑item tuples, etc.).

2. **Allocation (`tp_new` / `PyList_New`)** – Before asking `pymalloc` for bytes, the function checks the interpreter state's free list. If a dead object is available, it pops it, sets `ob_refcnt = 1`, and returns it.

3. **Deallocation (`tp_dealloc` / `list_dealloc`)** – When `ob_refcnt` hits 0, the function cleans up the object's payload (e.g., `DECREF`ing the items *inside* the list). Then, **instead of calling `PyObject_GC_Del()`**, it pushes the struct pointer onto the free list.

---

## 6. Important Data Structures

### `_PyFreeListState` (in modern CPython)
Inside `pycore_interp.h` or `pycore_freelist.h`, you will find a struct like:

```c
struct _freelist_state {
    PyListObject *lists[80];
    int numfree_lists;
    // ... free lists for dicts, floats, tuples, etc.
};
```

This ensures that memory recycling is strictly **per‑interpreter**.

---

## 7. Important Functions

| Function                     | Purpose                                                                                         |
|------------------------------|-------------------------------------------------------------------------------------------------|
| `PyList_New(size)`           | C API for `[]`. Contains the free‑list **pop** logic.                                           |
| `list_dealloc(op)`           | Called when a list dies. Contains the free‑list **push** logic.                                 |
| `PyTuple_New(size)`          | Tuple allocation. Tuples are immutable, so recycling them based on exact size is highly optimised. |

---

## 8. Important Macros

| Macro                           | Purpose                                                                             |
|---------------------------------|-------------------------------------------------------------------------------------|
| `_Py_FREELIST_POP(type, state)` | Internal macro to standardise popping from a free list.                             |
| `_Py_FREELIST_PUSH(type, state, obj)` | Internal macro to standardise pushing to a free list.                           |

---

## 9. Source Code Exploration

1. **`Objects/listobject.c`** – Search for `PyList_New`. At the very beginning you will see:
   ```c
   if (numfree > 0) {
       op = free_list[--numfree];
       // ... set ob_refcnt = 1, etc.
   } else {
       // ... call PyObject_GC_New
   }
   ```
   Notice how it avoids calling `PyObject_GC_New` entirely when a free object exists.

2. **`Objects/listobject.c`** – Search for `list_dealloc`. After it frees the array of items, you will see:
   ```c
   if (numfree < PyList_MAXFREELIST) {
       free_list[numfree++] = op;
   } else {
       Py_TYPE(op)->tp_free((PyObject *)op);
   }
   ```

3. **`Objects/tupleobject.c`** – Search for `PyTuple_New`. Observe the massive optimisation: it checks `size < PyTuple_MAXSAVESIZE` (usually 20), and routes to a specific free list based on the tuple's length.

---

## 10. Execution Flow – Tight Loop

Consider this Python code:

```python
for _ in range(10):
    a = [1]
```

**Step‑by‑step:**

1. **Iteration 1:** `[]` is created. Free list is empty. `pymalloc` allocates 24 bytes for the list struct.

2. **End of Iteration 1:** `a` is reassigned. The old list's refcount hits 0. `list_dealloc` runs. It frees the internal array `[1]` (calling `Py_DECREF` on the integer `1`), but **pushes the `PyListObject` struct** onto the free list.

3. **Iteration 2:** `[]` is created. `PyList_New` checks the free list, finds the list from Iteration 1, pops it, resets `ob_refcnt = 1`, and returns it.

4. **Result:** Over 10 iterations, `pymalloc` is only called **once** for the list struct. The exact same memory address is recycled 9 times.

---

## 11. Real Python Example – Observing Address Reuse

You can observe this from pure Python using `id()` (which returns the C memory address):

```python
a = []
addr_a = id(a)
del a

b = []
addr_b = id(b)

print(addr_a == addr_b)   # Prints: True (in CPython)
```

Because `a` was pushed to the free list, `b` immediately popped that exact same struct and memory address.

> **Note:** This is CPython‑specific behaviour. Other Python implementations (PyPy, Jython) may not reuse addresses so predictably.

---

## 12. Why This Design?

### Why bypass `pymalloc` if it's already O(1)?
`pymalloc` gives you raw bytes. To make those bytes a valid list, CPython must:
- Call `PyObject_GC_Track()` (register with the Garbage Collector).
- Set the `ob_type` pointer.
- Initialise the list‑specific fields.

When a list is put on a free list, it is **already tracked** by the GC and **already has its type set**. Bypassing this initialisation saves critical CPU cycles.

### Why is the limit so small (e.g., 80 for lists)?
If the free list was unbounded, creating 1,000,000 lists and deleting them would leave 1,000,000 dead structs in the free list forever. CPython caps it at **80** to balance:
- Rapid allocation of temporary variables.
- Long‑term memory bloat.

---

## 13. Common Beginner Mistakes

- **Mistake:** Benchmarking list creation using `timeit` and assuming it measures OS allocation speed.  
  **Correction:** Because `timeit` repeatedly creates and destroys in a loop, it is exclusively benchmarking the **Free List**, which is near‑instant. It does **not** reflect the cost of creating 100,000 lists simultaneously.

- **Mistake:** Assuming that when a script's RAM usage spikes, calling `gc.collect()` will return all memory to the OS.  
  **Correction:** Free lists hold onto memory. A dead float in the float free list is technically “free” to CPython, but to the Operating System, that memory is still actively held by the `python` process.

---

## 14. Summary

To maximise speed, CPython's core built‑in types (`list`, `dict`, `tuple`, `float`) maintain their own private “recycling bins” called **Free Lists**.

- When an object dies, its C struct is pushed to an array instead of being freed.
- When a new object is requested, it is popped from this array.
- This bypasses both `pymalloc` and GC initialisation, making temporary variable creation virtually cost‑free.

---

## 15. Mental Model to Remember

```
(User requests List) -> PyList_New()
                         │
                 [ Free List (Max 80) ]
                 /                    \
            (Empty)                  (Has Dead List)
              │                          │
        Call pymalloc()             Pop, wipe, return
      Init PyObject header               │
      Track in GC                        │
              │                          │
              └─────────► Done ◄─────────┘
```

---

## 16. Important Functions (Quick Reference)

| Function            | Purpose                                   |
|---------------------|-------------------------------------------|
| `PyList_New()`      | Allocates a new list – tries free list first. |
| `list_dealloc()`    | Destroys a list – pushes to free list if space. |
| `PyTuple_New()`     | Allocates a tuple – uses size‑specific free list. |

---

## 17. Important Structs

| Struct                 | Purpose                                                     |
|------------------------|-------------------------------------------------------------|
| `_PyFreeListState`     | Holds all free lists (lists, tuples, dicts, etc.) per interpreter. |

---

## 18. Important Files

| File                                | Role                                                         |
|-------------------------------------|--------------------------------------------------------------|
| `Objects/listobject.c`              | Free‑list logic for lists (pop/push in `PyList_New`/`list_dealloc`). |
| `Objects/tupleobject.c`             | Free‑list logic for tuples (multiple lists by size).         |
| `Include/internal/pycore_freelist.h` | Internal declarations for free‑list macros and structs.     |
| `Include/internal/pycore_interp.h`  | Where the `_PyFreeListState` lives (per‑interpreter).        |

---

## 19. Code‑Reading Exercises

1. Open `Objects/listobject.c`. Find `list_dealloc`. Verify the exact C `if` statement that checks the free list limit before falling back to `Py_TYPE(op)->tp_free`.

2. Open `Objects/tupleobject.c`. Find `tupledealloc`. Notice how it uses the tuple's size (`Py_SIZE(op)`) as an index into an array of free lists.

3. Open `Include/internal/pycore_interp.h` (or similar internal header depending on branch). Search for `free_list`. See how these are bound to the `PyInterpreterState` struct rather than being global variables.

---

## 20. Understanding Questions

1. If a list is stored in the free list, its `ob_refcnt` is 0. **Why doesn't the Cyclic Garbage Collector see it and try to destroy it** during a GC pass?  
   *(Hint: Think about what `tp_traverse` would do on a dead object.)*

2. Why do `tuples` have **20 different free lists** based on size, but `lists` only have **1 free list** regardless of how many items the list holds?  
   *(Hint: Think about where a tuple's payload lives vs where a list's payload lives in C memory.)*

3. If you write a C extension with a custom `PyTypeObject`, **do you get a free list automatically**, or must you implement the C array and push/pop logic yourself in your `tp_new`/`tp_dealloc`?

---

## 21. Suggested Next Files to Read

- **`Include/cpython/longintrepr.h`** – Now that we understand memory and allocation completely, we are ready to look at how specific Python data types are actually implemented in C.

---

## 22. Suggested Next Lesson

**Lesson 13 – Built‑in Types (int, float, bool)**  
We have laid all the memory foundations. It is time to look at the C structs that hold the actual data, starting with Python's infinite‑precision integers.
