# Lesson 10: The Cyclic Garbage Collector

We have covered Reference Counting, which handles 99% of memory management in CPython. But reference counting has a fatal mathematical flaw: **it cannot detect cycles**.

To fix this, CPython includes a secondary memory management system: the **Cyclic Garbage Collector**. As a C programmer, you must understand how CPython seamlessly injects a hidden linked-list into your object allocations to make this work.

---

## 1. Lesson Overview

In this lesson, we study the Cyclic Garbage Collector (GC). We learn how CPython:

- Tracks "container" objects (those that can hold references to others).
- Identifies isolated groups of objects that point only to each other (cycles).
- Safely tears them down without corrupting memory.

**Why it matters:**  
If you write a custom C extension type that holds references to other Python objects (like a custom tree node), you **must** integrate it with the GC. If you don't, any cycles involving your object will leak memory permanently.

**Prerequisites:** You deeply understand `PyObject` and `ob_refcnt` (Lessons 8 & 9).

---

## 2. Mental Model – The Cycle Detector

The GC is a **cycle detector**. It is **not** a tracing garbage collector like Java's or Go's, which scan from global roots.

- The GC maintains **doubly‑linked lists** of all objects *capable* of forming cycles (like lists, dicts, and custom classes).
- Integers and strings cannot hold pointers to other objects, so they are **not** tracked by the GC.

When the GC runs, it essentially asks:  
*"If I temporarily pretend all the internal references within this list don't exist, do any of these objects have a reference count greater than zero?"*

- **If yes** → Someone outside the cycle is holding it. It lives.
- **If no** → The entire group is a floating island of garbage. It dies.

```
Memory Layout of a GC-Tracked Object:

[ Hidden GC Header ] ◄── GC uses this to string objects into a linked list
[ PyObject Header  ] ◄── The normal Python pointer points HERE
[ Object Payload   ]
```

---

## 3. Where We Are in the Repository

| **Path / File**                   | **Role**                                                                    |
|-----------------------------------|-----------------------------------------------------------------------------|
| `Python/gc.c`                     | Core GC algorithm (cycle detection, collection logic).                      |
| `Include/internal/pycore_gc.h`    | Internal definitions – `PyGC_Head`, generation thresholds, etc.             |
| `Modules/gcmodule.c`              | Python‑facing API – the `gc` module (`gc.collect()`, `gc.disable()`, etc.). |
| `Include/objimpl.h`               | Public macros for GC allocation and tracking.                               |

**Why they matter:**  
Historically, the GC lived in `Modules/gcmodule.c`. In modern CPython, the core algorithm was moved to `Python/gc.c` (the internal VM implementation), leaving `Modules/gcmodule.c` just for the `import gc` Python‑facing API.

---

## 4. Concepts We Need First

### 4.1 Container Objects
Only objects that can contain references to *other* objects need cycle detection. Examples include lists, dicts, sets, tuples, and custom class instances. These are flagged with `Py_TPFLAGS_HAVE_GC`.

### 4.2 Generational GC Hypothesis
> "Most objects die young."

Scanning every list in memory is too slow. CPython divides objects into **3 generations** (Gen 0, 1, 2):

- New tracked objects start in **Gen 0**.
- If they survive a GC pass, they are promoted to Gen 1.
- Gen 0 is scanned **frequently**; Gen 2 is scanned **rarely**.

---

## 5. Architecture

1. **Allocation** – When a list is created, CPython calls `_PyObject_GC_Alloc()`. It allocates extra memory **before** the `PyObject` to hold a `PyGC_Head`.

2. **Tracking** – The object is added to the **Gen 0** doubly‑linked list via `PyObject_GC_Track()`.

3. **Trigger** – As allocations happen, a counter increments. If the counter exceeds a threshold (default ≈ 700 allocations without deallocations), a GC pass is **synchronously** triggered.

4. **Traversal** – The GC asks every object in the generation to yield all of its children using a C function pointer called **`tp_traverse`**.

5. **Cycle Detection** – By doing math on the reference counts *vs.* the number of incoming internal links, it isolates the garbage. This is the **trial deletion** algorithm.

6. **Clearing** – It calls **`tp_clear`** on the garbage objects to break the cycle (e.g., forcing a list to drop references to all its items), allowing standard reference counting to immediately free the memory.

---

## 6. Important Data Structures

### `PyGC_Head` (defined in `Include/internal/pycore_gc.h`)
The hidden header. In modern 64‑bit CPython, it is aggressively packed into just **two pointers (16 bytes)**:

```c
typedef struct _gc_head {
    struct _gc_head *_gc_next;   // Next object in the tracked list
    struct _gc_head *_gc_prev;   // Previous object
} PyGC_Head;
```

> **Note:** During collection, `_gc_prev` is temporarily repurposed to hold the object's `ob_refcnt` for the trial deletion algorithm.

---

## 7. Important Functions

| Function                       | Location         | Purpose                                                                                          |
|--------------------------------|------------------|--------------------------------------------------------------------------------------------------|
| `_PyObject_GC_Alloc(size)`     | `Python/gc.c`    | Wraps `malloc`. Allocates `size + sizeof(PyGC_Head)`. Returns a pointer shifted *forward* so it points to the `PyObject`. |
| `PyObject_GC_Track(op)`        | `Python/gc.c`    | Links the `PyGC_Head` into the current generation's linked list.                                 |
| `PyObject_GC_UnTrack(op)`      | `Python/gc.c`    | Unlinks the object from the list. **Crucial:** Must be called *before* destruction, otherwise the GC linked list becomes corrupted. |
| `gc_collect_main()`            | `Python/gc.c`    | The massive C function that performs the actual cycle detection and collection for a generation. |

---

## 8. Important Macros

| Macro                          | Purpose                                                                                              |
|--------------------------------|------------------------------------------------------------------------------------------------------|
| **`Py_VISIT(op)`**             | Used exclusively inside a type's `tp_traverse` function. When the GC asks a list what it contains, the list calls `Py_VISIT(item)` for every item. This macro tells the GC about the child object. |
| **`_Py_AS_GC(o)`**             | Pointer‑math macro that takes a `PyObject *` and subtracts `sizeof(PyGC_Head)` to find the hidden GC linked‑list node. |

---

## 9. Source Code Exploration

1. **`Include/internal/pycore_gc.h`** – Search for `PyGC_Head`. Note how small it is – just two pointers.

2. **`Include/objimpl.h`** – Search for `#define _Py_AS_GC(o)`. This is the pointer‑math macro:
   ```c
   #define _Py_AS_GC(o) ((PyGC_Head *)(((char *)(o)) - sizeof(PyGC_Head)))
   ```

3. **`Python/gc.c`** – Search for `subtract_refs()`. This is the genius of the algorithm. It iterates over every object, calls `tp_traverse` to find its children, and *decrements* a temporary copy of the child's refcount. If the temporary refcount hits 0, the object is *only* referenced by other objects in this GC generation.

---

## 10. Execution Flow – A Self‑Referential List

Let's trace: `a = []; a.append(a); del a`

1. **`a = []`**  
   `_PyObject_GC_Alloc` creates the list. `PyObject_GC_Track` puts it in Gen 0. Refcount = 1.

2. **`a.append(a)`**  
   The list's internal array now points to the list itself. Refcount becomes **2** (one from variable `a`, one from the list's own array).

3. **`del a`**  
   The variable `a` is destroyed. `Py_DECREF` is called. Refcount drops to **1**.

4. **Memory Leak!**  
   The list is inaccessible from Python, but its refcount is 1 (held by itself). Reference counting alone cannot free it.

5. **GC Trigger** – Later, the Gen 0 threshold is reached. `gc_collect_main()` runs.

6. **Trial Deletion** – The GC copies the list's refcount (1) to a temporary `gc_refs` field. It calls the list's `tp_traverse`. The list reports: "I point to myself." The GC decrements the temporary refcount → it hits 0.

7. **Cycle Detected** – The GC concludes this list is garbage.

8. **Clearing** – The GC calls the list's `tp_clear`. `tp_clear` empties the list's internal array. The internal self‑reference is dropped, triggering `Py_DECREF` on the list itself.

9. **Free** – Refcount hits 0. The list is naturally deallocated.

---

## 11. Real Python Example – Custom Class Cycle

```python
import gc

class Node:
    pass

node1 = Node()
node2 = Node()
node1.child = node2
node2.parent = node1

del node1, node2
print(gc.collect())   # Manually trigger GC. Prints 2 (the number of objects collected).
```

Because custom classes use `__dict__` to store attributes, and dicts are tracked containers, `node1` and `node2` form a cycle that refcounting cannot free. `gc.collect()` explicitly runs `gc_collect_main()`.

---

## 12. Why This Design?

### Why use a hidden prepended header (`PyGC_Head`)?
Memory layout optimisation. If `PyObject` contained the `next`/`prev` pointers, **every** object (including millions of ints and strings) would be 16 bytes larger. By prepending it *only* for GC‑tracked allocations, CPython saves massive amounts of memory.

### Why `tp_traverse` and `tp_clear`?
The C compiler doesn't know where a custom C struct hides its `PyObject *` pointers. The GC cannot blindly scan memory like C/C++ conservative collectors do. CPython relies on **cooperative types** – the type itself must provide a function (`tp_traverse`) that explicitly hands its children to the GC.

---

## 13. Common Beginner Mistakes

- **Mistake:** "Python has a Garbage Collector, so I don't need to worry about `Py_DECREF`."  
  **Correction:** The GC is **only** a backup for cycles. If you `Py_INCREF` an object and forget to `Py_DECREF` it, its refcount will *always* be artificially high. The GC will assume it is safely in use and will **never** clean it up.

- **Mistake:** Thinking the GC runs on a background OS thread.  
  **Correction:** CPython's GC is completely **synchronous**. It pauses the main interpreter loop during a bytecode instruction allocation to scan memory. If Gen 2 becomes massive, this pause can cause latency spikes in servers.

- **Mistake:** Not calling `PyObject_GC_UnTrack()` in `tp_dealloc`.  
  **Correction:** If you `free()` a node without removing it from the GC linked list, the GC's `next` pointer will point to freed memory, causing a crash on the next collection.

---

## 14. Summary

The Cyclic Garbage Collector supplements Reference Counting to clean up isolated reference cycles. It operates only on container objects, linking them into generations using a hidden `PyGC_Head` struct placed **immediately before** the `PyObject` header in memory. The GC detects cycles using the **trial deletion** algorithm (subtracting internal references) and breaks them via `tp_clear`.

---

## 15. Mental Model to Remember

```
(Memory Address X)
  │
  ▼
┌──────────────────┐
│ PyGC_Head        │ ◄── GC walks this doubly-linked list
│  _gc_next        │
│  _gc_prev        │
├──────────────────┤
│ PyObject Header  │ ◄── Python variables point here
│  ob_refcnt       │
│  ob_type         │
├──────────────────┤
│ Payload (Array)  │ ◄── tp_traverse looks here to find children
└──────────────────┘
```

---

## 16. Important Functions (Quick Reference)

| Function                 | Purpose                                              |
|--------------------------|------------------------------------------------------|
| `_PyObject_GC_Alloc()`   | Allocates memory with a hidden `PyGC_Head` prefix.   |
| `PyObject_GC_Track()`    | Adds an object to the GC generation list.            |
| `PyObject_GC_UnTrack()`  | Removes an object from the GC list (must be called before `tp_dealloc`). |
| `gc_collect_main()`      | The main cycle‑detection routine.                    |

---

## 17. Important Structs

| Struct        | Purpose                                                       |
|---------------|---------------------------------------------------------------|
| `PyGC_Head`   | Hidden header – `next`/`prev` pointers for the GC linked list. |

---

## 18. Important Files

| File                         | Role                                                         |
|------------------------------|--------------------------------------------------------------|
| `Python/gc.c`                | Core GC algorithm – collection, traversal, allocation.       |
| `Include/internal/pycore_gc.h` | Internal GC structures (`PyGC_Head`, generation states).   |
| `Include/objimpl.h`          | Public macros (`_Py_AS_GC`, `PyObject_GC_Track`).            |
| `Modules/gcmodule.c`         | Python `gc` module implementation (`gc.collect()`, `gc.disable()`). |

---

## 19. Code‑Reading Exercises

1. Open `Include/objimpl.h`. Search for `#define _Py_AS_GC(o)`. Observe the pointer casting: it casts `o` to `char *`, subtracts `sizeof(PyGC_Head)`, and casts it to `PyGC_Head *`.

2. Open `Python/gc.c`. Find `subtract_refs()`. Look at how it loops over the generation, calls `traverse()`, and modifies the `gc_refs` count.

3. Open `Objects/listobject.c`. Find `list_traverse()`. See how it loops over `ob_item[i]` and calls `Py_VISIT()` on each element. This is how the list cooperates with the GC.

---

## 20. Understanding Questions

1. Why must an object call `PyObject_GC_UnTrack()` **before** its memory is freed in `tp_dealloc`?  
   *(Hint: What happens to a linked list if you `free()` a node without removing it?)*

2. If an object is immutable and only contains references to other immutable objects (like a tuple of ints), **can it ever form a cycle? Should it be tracked by the GC?**

3. If a C extension creates a custom struct holding a `PyObject *` but fails to implement `tp_traverse`, **what will happen** if that struct participates in a cycle?

---

## 21. Suggested Next Files to Read

- **`Objects/obmalloc.c`** – We know how objects are allocated conceptually, but we need to see exactly where the bytes come from.

---

## 22. Suggested Next Lesson

**Lesson 11 – Memory Management (pymalloc)**  
We understand refcounting and cycle detection. Now we hit the bare metal: how does CPython avoid thrashing the OS `malloc` when it needs to create 10,000 tiny integer objects?
