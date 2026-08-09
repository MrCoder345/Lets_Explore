# Lesson 11: Memory Management – pymalloc

You have seen how objects are tracked and destroyed. Now we hit the bare metal: **where does the memory actually come from?**

As a C programmer, you know that calling `malloc()` and `free()` for every 24‑byte struct is a performance disaster. It causes severe heap fragmentation, and the hidden bookkeeping overhead of `malloc` (often 8–16 bytes per allocation) doubles your memory footprint.

Since Python programs create and destroy millions of tiny objects (like ints, floats, and strings), CPython bypasses the OS `malloc` entirely for small objects and uses its own specialised memory manager called **`pymalloc`**.

---

## 1. Lesson Overview

In this lesson, we study `pymalloc`, CPython's highly optimised small‑object memory allocator. We explore its **3‑tier hierarchical architecture** (Arenas, Pools, and Blocks) and see how it completely bypasses the system allocator for any object under 512 bytes.

**Why it matters:**  
- If you allocate C memory in an extension using the wrong API, you will crash the VM.  
- Understanding `pymalloc` is crucial for optimising Python memory usage.  
- It also explains how modern CPython handles the transition to free‑threaded (No‑GIL) execution.

**Prerequisites:** You understand C memory alignment, fragmentation, and basic OS‑level memory pages.

---

## 2. Mental Model – The Real Estate Developer

Think of `pymalloc` as a real estate developer. It doesn't build individual houses (`malloc`ing a single object). Instead:

- It buys a massive tract of land (an **Arena**).
- Subdivides it into identical apartment buildings (**Pools**).
- Splits those buildings into identical rooms (**Blocks**).

When CPython needs 28 bytes for a float, `pymalloc`:

1. Rounds 28 up to the nearest 8‑byte boundary (32 bytes).
2. Looks for a **Pool** specifically designated for 32‑byte **Blocks**.
3. Hands you a pointer to an empty 32‑byte Block in **O(1)** time via a singly‑linked free list.

```
[ Arena (256 KB - allocated via mmap/malloc) ]
  ├── [ Pool (4 KB) - For 8-byte objects  ] ──> [Block][Block][Block]...
  ├── [ Pool (4 KB) - For 32-byte objects ] ──> [Block][Block][Block]...
  └── [ Pool (4 KB) - For 64-byte objects ] ──> [Block][Block][Block]...
```

---

## 3. Where We Are in the Repository

| **Path / File**           | **Role**                                                                      |
|---------------------------|-------------------------------------------------------------------------------|
| `Objects/obmalloc.c`      | Complete implementation of the `pymalloc` architecture.                       |
| `Include/cpython/pymem.h` | Public C API for allocating memory (the three domains: Raw, Mem, Object).     |

**Why they matter:**  
- `pymem.h` defines the public C API for allocating memory (the domains).  
- `obmalloc.c` contains the entire, massive implementation of the `pymalloc` architecture – one of the most heavily commented files in CPython.

---

## 4. Concepts We Need First

### Allocator Domains
CPython does **not** have one single memory allocator; it has three distinct “domains”:

| Domain          | API Entry Point          | Purpose                                                                                       |
|-----------------|--------------------------|-----------------------------------------------------------------------------------------------|
| **Raw Domain**  | `PyMem_RawMalloc()`      | Wrapper around system `malloc()`. Used for OS‑level buffers or memory needed *before* the VM initialises. Thread‑safe by default (relies on OS). |
| **Mem Domain**  | `PyMem_Malloc()`         | Used for generic C structures tied to the Python runtime (e.g., internal buffers).            |
| **Object Domain** | `PyObject_Malloc()`    | Used for allocating `PyObject` structs. **This is where `pymalloc` lives.**                   |

---

## 5. Architecture – The Three Tiers

The `pymalloc` allocator handles allocations **≤ 512 bytes** (which covers >99% of Python objects). If an allocation is > 512 bytes, `pymalloc` instantly delegates it to the system `malloc()`.

### 5.1 Blocks
- The actual chunk of memory returned to the caller.
- Sizes are multiples of **8 bytes**: 8, 16, 24, ... 512.
- There are exactly **64 “size classes”** (512 / 8).

### 5.2 Pools
- A **4 KB** chunk of memory (typically one OS page).
- A pool contains Blocks of **only one size class**.
  - Example: a pool dedicated to 32‑byte blocks.
- Pools have a `freeblock` pointer, functioning as a fast LIFO stack of available blocks.

### 5.3 Arenas
- A **256 KB** chunk of memory requested directly from the OS (`mmap` or `malloc`).
- An Arena is simply a container for **64 Pools** (256 KB / 4 KB).

---

## 6. Important Data Structures

### `pool_header` (defined in `Objects/obmalloc.c`)
```c
struct pool_header {
    union { 
        uint8_t used;           // Number of blocks in use
        uint8_t ref_count;      // Same field, different name
    };
    uint8_t szidx;              // Size class index (0–63)
    uint8_t nextoffset;         // Offset to next free block
    struct pool_header *nextpool;
    struct pool_header *prevpool;
    uint8_t *freeblock;         // Pointer to the first free block
    // ... more fields for alignment
};
```

- `szidx` determines the block size (e.g., `3` → 32 bytes).
- `freeblock` is a linked list of available blocks **stored inside the unused blocks themselves** (saving memory).
- `nextpool`/`prevpool` link pools of the same size class together.

### `arena_object`
Tracks the usage of a 256 KB arena. Contains a pointer to the first pool and the number of free pools.

---

## 7. Important Functions

| Function                       | Purpose                                                                                   |
|--------------------------------|-------------------------------------------------------------------------------------------|
| `PyObject_Malloc(size_t size)` | Entry point. If `size <= 512`, finds the right pool and returns a block. Else, calls system `malloc()`. |
| `PyObject_Free(void *p)`       | Figures out if `p` points inside a `pymalloc` Arena. If yes, returns the block to the pool's free list. If no, calls system `free()`. |

---

## 8. Important Macros

| Macro                        | Value          | Purpose                                                       |
|------------------------------|----------------|---------------------------------------------------------------|
| `ALIGNMENT`                  | `8`            | All block sizes are rounded up to this (8‑byte alignment).    |
| `SMALL_REQUEST_THRESHOLD`    | `512`          | Allocations ≤ this size go through `pymalloc`.                |

---

## 9. Source Code Exploration

1. **`Objects/obmalloc.c`** – This is one of the most **heavily commented** files in CPython.  
   - Search for `struct pool_header` and observe the fields.
   - Notice how the free list pointers are stored **inside** the unused blocks themselves to save space!

2. **`Objects/obmalloc.c`** – Search for `pymalloc_alloc(void *ctx, size_t nbytes)`. This is the core logic. You will see bitwise math used to convert `nbytes` into a `size class index`.

3. **`Include/cpython/pymem.h`** – Look for the declarations of `PyObject_Malloc` and `PyObject_Free`.

---

## 10. Execution Flow – Allocating a Float

Let's trace: `x = 42` (requires a 28‑byte `PyLongObject` in C):

1. Python VM calls `PyObject_Malloc(28)`.
2. `obmalloc.c` rounds 28 up to **32 bytes** (size class index `3`).
3. It checks an array of `usedpools[3]` to find a 4 KB pool that:
   - Dispenses 32‑byte blocks.
   - Has empty space.
4. It pops the first available 32‑byte block off the pool's `freeblock` list.
5. It returns the C pointer.

**Cost:** A few pointer dereferences – **O(1)** time, **zero syscalls**.

---

## 11. Real Python Example – 1 Million Floats

```python
objects = [float(i) for i in range(1000000)]
```

If CPython used system `malloc`:
- 1 million syscalls.
- 16 bytes of hidden `malloc` metadata per object → ~16 MB extra overhead.

Instead, `pymalloc`:
- Requests a few dozen 256 KB Arenas from the OS.
- Neatly packs the floats into 4 KB Pools.
- **No `malloc` metadata per object** → saves ~16 MB of RAM in this single operation.

---

## 12. Why This Design?

### Why use 3 tiers (Arena, Pool, Block)?
- If a Pool (4 KB) becomes completely empty, it can be **repurposed** for a different size class.
- But CPython can only return memory to the OS at the **Arena** (256 KB) level. If an entire Arena becomes 100% empty, CPython calls `free()` on it, giving RAM back to the OS.

### Concurrency Context (Crucial)
- Historically, `pymalloc` is **completely thread‑unsafe**. It relies on the **Global Interpreter Lock (GIL)** to prevent two threads from corrupting the free lists.
- **In Python 3.13+ (free‑threaded builds):** Because the GIL can be disabled, `pymalloc` cannot be used as‑is. Under `--disable-gil`, CPython entirely bypasses `pymalloc` and relies on **`mimalloc`** (a Microsoft‑developed thread‑local allocator) to manage small objects safely across multiple threads!

---

## 13. Common Beginner Mistakes

- **Mistake:** Using `PyObject_Malloc()` to allocate memory, but using `free()` to release it.  
  **Correction:** `free()` expects a system memory pointer. If you pass it a pointer inside a `pymalloc` Pool, the OS allocator will instantly segfault because it cannot find its hidden `malloc` metadata. **Always match the allocator domain** – `PyObject_Malloc` goes with `PyObject_Free`.

- **Mistake:** Assuming Python memory always drops when `del` is called.  
  **Correction:** Due to fragmentation, an Arena is only returned to the OS if **every single block** inside its 64 Pools is freed. A single surviving 24‑byte float can hold a 256 KB Arena hostage in memory indefinitely.

---

## 14. Summary

CPython manages small objects (≤ 512 bytes) using `pymalloc`, an allocator optimised to prevent fragmentation and overhead:

- Carves memory into **256 KB Arenas**, **4 KB Pools**, and **8‑byte aligned Blocks**.
- Operates in **O(1)** time and avoids system syscalls.
- In standard builds, it relies on the **GIL** for thread safety.
- In free‑threaded builds (3.13+), CPython swaps it out for **`mimalloc`**.

---

## 15. Mental Model to Remember

```
System OS
   └── Arena (256KB) - Returned to OS only if 100% empty
         └── Pool (4KB) - Dedicated to one size (e.g., 32-bytes)
               ├── Block (32b) - [ In Use: Float 1.2 ]
               ├── Block (32b) - [ In Use: Float 3.4 ]
               └── Block (32b) - [ FREE ] ◄── freeblock pointer
```

---

## 16. Important Functions (Quick Reference)

| Function                 | Purpose                                               |
|--------------------------|-------------------------------------------------------|
| `PyObject_Malloc(size)`  | Allocate memory from `pymalloc` (or fallback to system). |
| `PyObject_Free(ptr)`     | Free memory allocated by `PyObject_Malloc`.           |

---

## 17. Important Structs

| Struct          | Purpose                                               |
|-----------------|-------------------------------------------------------|
| `pool_header`   | Tracks a 4 KB pool – its size class, free list, etc.  |
| `arena_object`  | Tracks a 256 KB arena and its pools.                  |

---

## 18. Important Files

| File                       | Role                                                                  |
|----------------------------|-----------------------------------------------------------------------|
| `Objects/obmalloc.c`       | Complete `pymalloc` implementation – heavily commented.               |
| `Include/cpython/pymem.h`  | Public C API – `PyObject_Malloc`, `PyObject_Free`, and domain macros. |

---

## 19. Code‑Reading Exercises

1. Open `Objects/obmalloc.c`. Read the **massive block comment at the very top**. It brilliantly explains the Arena/Pool/Block architecture.

2. In the same file, locate `#define SMALL_REQUEST_THRESHOLD`. Confirm it is set to `512`.

3. Search for `pymalloc_alloc`. Look for the fallback condition: if the size exceeds the threshold, it directly returns `PyMem_RawMalloc(nbytes)`.

4. Find `pool_header` and observe how `freeblock` is stored – a simple pointer to the next free block.

---

## 20. Understanding Questions

1. If an object is exactly **17 bytes**, how large will its `pymalloc` Block be?

2. If a C extension allocates a **1 MB string**, will that allocation live in a `pymalloc` Arena or on the system heap?

3. Why does `pymalloc` store its free list pointers **inside the memory of the freed blocks themselves**, rather than allocating a separate C array to track them?

---

## 21. Suggested Next Files to Read

- **`Objects/listobject.c`** or **`Objects/dictobject.c`** – To see how specific types implement an *additional* layer of memory caching on top of `pymalloc` (called **Free Lists**).

---

## 22. Suggested Next Lesson

**Lesson 12 – Object Allocation & Free Lists**  
We now know how `pymalloc` provides raw C bytes fast. But initialising `PyObject` structs still takes work. Next, we will see how built‑in types aggressively recycle dead objects to avoid even touching `pymalloc`.
